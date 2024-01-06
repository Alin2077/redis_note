## 开场白
1. 一些问题
    * Redis的跳跃列表了解吗？有什么缺点？
    * redis项目中如何使用各种数据？
    * redis的多路io复用如何理解，为什么单线程还可以抗那么高的qps？
    * redis的zset的底层实现，(压缩列表和跳表)，这样设计的优缺点。
2. Redis数据类型的底层数据结构
    * SDS动态字符串
    * 双向链表
    * 压缩列表ziplist
    * 哈希表hashtable
    * 跳表skiplist
    * 整数集合intset
    * 快速列表quicklist
    * 紧凑列表listpack
## 六大类型
1. string、bitmap、hyperLoglog
2. list
3. hash
4. set
5. zset、GEO
6. stream
## redisObject
1. Redis定义的一种结构体，用来表示string、list、hash、set、zset这些数据类型
2. Redis中每个对象都是一个redisObject结构
3. 字典、KV是什么？（重点）
    * 每个键值对都会有一个dictEntry （位于源码文件dict.c或dict.h中）
        ```c
        struct dictEntry { /* 表示哈希表节点的机构。存放了key和value的指针 */
            void *key; 
            union {
                void *val;
                uint64_t u64;
                int64_t s64;
                double d;
            } v;
        struct dictEntry *next;     /* Next entry in the same hash bucket. */
        };
        /* dictEntry内部存放的是指向key和value的指针，而key和value都是单独的对象，由redisObject构成 */
        ```
    * 如何从dictEntry到redisObject？
        1. 查看redis源码中定义的redisObject（位于源码文件server.h中）
            ```c
            struct redisObject {
                unsigned type:4; /* 对象类型，如OBJ_LIST */
                unsigned encoding:4; /* 具体的底层数据结构 */
                unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                                        * LFU data (least significant 8 bits frequency
                                        * and most significant 16 bits access time). */
                int refcount; /* 当前被引用多少次 */
                void *ptr;  /* 指向更底层的结构，SDS、双向链表、压缩列表、哈希表、跳表、整数集合等 */
            };
            ```
4. redisObject与redis数据类型、redis所有编码方式（底层实现）三者之间的关系
    * redisObject结构体中的type属性即表示数据类型
    * redisObject结构体中的encoding属性即表示编码方式（底层实现）
## 五大结构底层C语言源码分析
0. 大纲
    * string = SDS
    * set = intset + hashtable
    * zset = skiplist + ziplist || zset = skiplist + listpack
    * list = quicklist + ziplist || list = quicklist
    * hash = hashtable + ziplist || hash = hashtable + listpack
1.  string类型
    1. 三大物理编码方式
        * int
            * 保存long型（长整型）的64位（8个字节）有符号整数  ---> 超过长度直接转化成embstr或raw
            * 最小值为 -2的63次方
            * 最大值为 2的63次方
            * 只有整数会使用int，浮点数在redis中会转化成字符串的值
        * embstr
            * 代表enmstr格式的SDS（Simple Dynamic String 简单动态字符串），保存长度小于44字节的字符串
        * raw
            * 保存长度大于44字节的字符串
    2. 为什么设计一个SDS？
        * 先看源码（位于sds.h内）
            ```c
            typedef char *sds;
            /* Note: sdshdr5 is never used, we just access the flags byte directly.
            * However is here to document the layout of type 5 SDS strings. */
            struct __attribute__ ((__packed__)) sdshdr5 {
                unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
                char buf[];
            };
            struct __attribute__ ((__packed__)) sdshdr8 {
                uint8_t len; /* used */
                uint8_t alloc; /* excluding the header and null terminator */
                unsigned char flags; /* 3 lsb of type, 5 unused bits */
                char buf[];
            };
            struct __attribute__ ((__packed__)) sdshdr16 {
                uint16_t len; /* used */
                uint16_t alloc; /* excluding the header and null terminator */
                unsigned char flags; /* 3 lsb of type, 5 unused bits */
                char buf[];
            };
            struct __attribute__ ((__packed__)) sdshdr32 {
                uint32_t len; /* used */
                uint32_t alloc; /* excluding the header and null terminator */
                unsigned char flags; /* 3 lsb of type, 5 unused bits */
                char buf[];
            };
            struct __attribute__ ((__packed__)) sdshdr64 {
                uint64_t len; /* used */
                uint64_t alloc; /* excluding the header and null terminator */
                unsigned char flags; /* 3 lsb of type, 5 unused bits */
                char buf[];
            };
            /* len当前字符数组的长度 alloc当前字符数组总共分配的内存大小 flags 当前字符数组的属性 buf[] 字符串真正的值 */
            ```
        * len 使得在获取长度是无需遍历字符数组，时间复杂度O(1)
        * alloc 可以用来计算free也就是字符串已经分配的未使用空间，有了这个值就可以引入预分配空间的算法了，而不是去考虑内存分配的问题
        * alloc的值有两种情况，如果实际字符串小于1M，则alloc值是字符串长度；如果实际字串大于1M，则alloc就是1M
        * 因为不需要使用"\0"做结束符，所以二进制安全
    3. 向redis中存入string类型数据，内部所作的步骤（不必死扣源码，但要了解过程）
        1. 先判断需要转化成哪种物理编码格式
        2. 根据选定的物理编码格式执行具体的操作
            1. int编码
                * Redis在启动时会预先建立10000个分别存储0~9999的redisObject变量作为共享变量，这就意味着如果set字符串的键值在0~10000之间的话，可以直接指向共享对象而不需要重新建立新对象
                * 存储时先做判断：字符串长度<=20且可以转换成long型
                * redisObject中的ptr属性直接存储数字，不作为指针节省空间开销
            2. embstr编码
                * 不能转换成long型并且长度小于44
                * embedded string，表示嵌入式的String。从内存结构上来讲，即字符串sds结构体与其对应的redisObject对象分配在同一块连续的内存空间，字符串sds像是嵌入在redisObject中一样 ---- 在源码中可以看到 ptr=sh+1，即sds紧跟在redisObject后
                * 只读的。因此如果要修改已经存入的embstr，必须将其改为raw格式
            3. raw编码
                * 长度大于44即转换成raw格式，或者修改后的embstr
                * 和embstr不同的是，raw格式的内存空间需要单独申请
