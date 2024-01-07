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
                * 不能转换成long型并且长度小于44t
                * embedded string，表示嵌入式的String。从内存结构上来讲，即字符串sds结构体与其对应的redisObject对象分配在同一块连续的内存空间，字符串sds像是嵌入在redisObject中一样 ---- 在源码中可以看到 ptr=sh+1，即sds紧跟在redisObject后
                * 只读的。因此如果要修改已经存入的embstr，必须将其改为raw格式
            3. raw编码
                * 长度大于44即转换成raw格式，或者修改后的embstr
                * 和embstr不同的是，raw格式的内存空间需要单独申请
2. hash类型
    1. 两种编码格式
        * ziplist + hashtable （redis6）
            * 在redis命令行使用 config get hash*，可得出下列结果
                ```shell
                1) "hash-max-ziplist-entries"
                2) "512"
                3) "hash-max-ziplist-value"
                4) "64"
                ```
            * hash-max-ziplist-entris：使用压缩列表保存时哈希集合中的最大元素个数
            * hash-max-ziplist-value：使用压缩列表保存时哈希集合中单个元素的最大长度
            * hash类型键的字段个数<font color="red">小于</font>hash-max-ziplist-entries并且每个字段名和字段值的长度<font color="red">小于</font>hash-max-ziplist-value时，会以ziplist存储数据
            * 当不满足ziplist的配置条件时，redis会以hashtable存储数据
            * ziplist升级到hashtable可以，反过来降级不可以
        * listpack + hashtable （redis7）
            * 在redis命令行使用 config get hash*，可得出下列结果
                ```shell
                1) "hash-max-ziplist-entries"
                2) "512"
                3) "hash-max-ziplist-value"
                4) "64"
                5) "hash-max-listpack-entries"
                6) "512"
                7) "hash-max-listpack-value"
                8) "64"
                ```
    2. hashtable编码格式解读
        * 在Redis中，hashtable被称为字典（dictionary），它是一个数组+链表的结构
        * OBJ_ENCODING_HT这种编码方式内部才是真正的哈希表结构，或称为字典结构，其可以实现O(1)复杂度的读写操作，因此效率很高
        * 在Redis内部，从OBJ_ENCODING_HT类型到底层的散列表数据结构是一层层嵌套下去的，如下：
            ```shell
            OBJ_ENCODING_HT(宏观的HT编码) --> dict(字典) --> dictht(哈希表) --> dictEntry(哈希节点)
            ```
    3. ziplist编码格式解读
         * Ziplist压缩列表是一种紧凑编码格式，总体思想是多花时间来换取节约空间，即以部分读写性能为代价，来换取极高的内存空间利用率，因此<font color="red">只会用于字段少，且字段值也较小的场景</font>。
         * 压缩列表内存利用率极高的原因与其连续内存的特性是分不开的
         * 为了节约内存而开发，它是由连续内存块组成的顺序数据结构，有点类似于数组
         * ziplist是一个经过特殊编码的双向链表，<font color="red">它不存储指向前一个链表节点的prev和指向下一个链表节点的指针next</font>而是<font color="red">存储上一个节点长度和当前节点长度</font>，经过牺牲部分读写性能，来换取高效的内存空间利用率，节约内存，是一种时间换空间的思想。只用在<font color="red">字段个数少，字段值小的场景里面</font>
         * zlbytes|zltail|zllen|entry1|entry2|...|entryN|elend
            * zlbytes：压缩列表占用内存字节数 uint32_t类型，占4字节 
            * zltail：尾节点至起始节点的偏移量 uint32_t类型，占4字节 
            * zlen：节点数量 uint16_t类型，占2字节,如果长度超过了65535，节点的真实数量需要遍历整个ziplist才可获得
            * entryN：压缩列表包含的各个节点，节点长度由节点内容确定
            * zlend：特殊值0下FF，标记压缩列表末端 uint8_t类型，占1字节 
        * zlentry，压缩列表节点的构成
            ```c
            /* We use this function to receive information about a ziplist entry.
            * Note that this is not how the data is actually encoded, is just what we
            * get filled by a function in order to operate more easily. */
            typedef struct zlentry {
                unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
                unsigned int prevrawlen;     /* Previous entry len. */
                unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                                For example strings have a 1, 2 or 5 bytes
                                                header. Integers always use a single byte.*/
                unsigned int len;            /* Bytes used to represent the actual entry.
                                                For strings this is just the string length
                                                while for integers it is 1, 2, 3, 4, 8 or
                                                0 (for 4 bit immediate) depending on the
                                                number range. */
                unsigned int headersize;     /* prevrawlensize + lensize. */
                unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                                the entry encoding. However for 4 bits
                                                immediate integers this can assume a range
                                                of values and must be range-checked. */
                unsigned char *p;            /* Pointer to the very start of the entry, that
                                                is, this points to prev-entry-len field. */
            } zlentry;
            ```
    4. listpack编码方式解读
        * 参数配置和ziplist通用
        * 升级、降级和ziplist类似
        * 一个空的listpack是7个字节，其中4个字节记录listpack的总字节数，2个字节记录listpack的元素数量，1个字节标识结束（0xFF）
        * listpack结构
            * listpack由4部分构成，分别为Total Bytes、num-elements、elementEntry、listpack-end-byte
            * entry的结构
                1. 当前元素的编码类型（entry-encoding）
                2. 元素数据（entry-data）
                3. 编码类型和元素数据这两部分的长度（entry-len）
                ```c
                /* Each entry in the listpack is either a string or an integer. */
                typedef struct {
                    /* When string is used, it is provided with the length (slen). */
                    unsigned char *sval;
                    uint32_t slen;
                    /* When integer is used, 'sval' is NULL, and lval holds the value. */
                    long long lval;
                } listpackEntry;
                ```吧
    5. 为什么将ziplist改成listpack？
        1. ziplist的缺点
            * 如果前一个节点的长度小于254字节，那么prevlen属性需要用1字节的空间来保存
            * 如果前一个节点的长度大于等于254字节，那么prevlen属性需要用5字节的空间来保存
            * 会出现连锁更新问题 ****
                1. 假设一个压缩列表中由多个连续的、长度在250~253之间的节点；此时每个节点的prevlen属性需要用1字节的空间来保存这个长度值
                2. 如果将一个长度大于等于254字节的新节点加入到压缩列表的表头节点，即新节点将成为entry1的前置节点；此时就需要对压缩列表的空间重分配操作并将ebtry1节点的prevlen属性从原来的1字节扩展为5字节
                3. entry1的节点原来为250~253节点，因为prevlen属性的扩展，导致总字节长度大于254，后续的每个类似的节点都需要更改prevlen属性
3. list类型
    1. 两种编码格式
        1. quicklist（redis6）内部是ziplist
            * 在redis命令行执行 config get list* 命令
                ```shell
                1) "list-max-ziplist-size"
                2) "-2"
                3) "list-compress-depth"
                4) "0"
                ```
            * list-compress-depth：表示一个quicklist两端不被压缩的节点个数。这里的节点是指双向链表的节点，而不是指ziplist里面的数据项个数；0是特殊值，表示都不压缩
            * list-max-ziplist-size：当取正值时，表示按照数据项个数来限定每个quicklist节点上的ziplist长度。比如，当这个参数配置成5的时候，表示每个quicklist节点的ziplist最多包含5个数据项；<font color="red">当取负值时，表示按照占用字节数来限定每个quicklist节点上的ziplist长度</font>。当取负值时，只能取-1到-5这五个值：
                1. -5：每个quicklist节点上的ziplist大小不能超过 64 kb
                2. -4：每个quicklist节点上的ziplist大小不能超过 32 kb
                3. -3：每个quicklist节点上的ziplist大小不能超过 16 kb
                4. -2：每个quicklist节点上的ziplist大小不能超过 8 kb
                5. -1：每个quicklist节点上的ziplist大小不能超过 4 kb
        2. list用quicklist来存储，quicklist存储了一个双向链表，每个节点内部数据都是一个ziplist
            ```c
            /* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
            * 'count' is the number of total entries.
            * 'len' is the number of quicklist nodes.
            * 'compress' is: 0 if compression disabled, otherwise it's the number
            *                of quicklistNodes to leave uncompressed at ends of quicklist.
            * 'fill' is the user-requested (or default) fill factor.
            * 'bookmarks are an optional feature that is used by realloc this struct,
            *      so that they don't consume memory when not used. */
            typedef struct quicklist {
                quicklistNode *head;
                quicklistNode *tail;
                unsigned long count;        /* total count of all entries in all listpacks */
                unsigned long len;          /* number of quicklistNodes */
                signed int fill : QL_FILL_BITS;       /* fill factor for individual nodes */
                unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
                unsigned int bookmark_count: QL_BM_BITS;
                quicklistBookmark bookmarks[];
            } quicklist;

            /* quicklistNode is a 32 byte struct describing a listpack for a quicklist.
            * We use bit fields keep the quicklistNode at 32 bytes.
            * count: 16 bits, max 65536 (max lp bytes is 65k, so max count actually < 32k).
            * encoding: 2 bits, RAW=1, LZF=2.
            * container: 2 bits, PLAIN=1 (a single item as char array), PACKED=2 (listpack with multiple items).
            * recompress: 1 bit, bool, true if node is temporary decompressed for usage.
            * attempted_compress: 1 bit, boolean, used for verifying during testing.
            * dont_compress: 1 bit, boolean, used for preventing compression of entry.
            * extra: 9 bits, free for future use; pads out the remainder of 32 bits */
            typedef struct quicklistNode {
                struct quicklistNode *prev;
                struct quicklistNode *next;
                unsigned char *entry;
                size_t sz;             /* entry size in bytes */
                unsigned int count : 16;     /* count of items in listpack */
                unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
                unsigned int container : 2;  /* PLAIN==1 or PACKED==2 */
                unsigned int recompress : 1; /* was this node previous compressed? */
                unsigned int attempted_compress : 1; /* node can't compress; too small */
                unsigned int dont_compress : 1; /* prevent compression of entry that will be used later */
                unsigned int extra : 9; /* more bits to steal for future usage */
            } quicklistNode;
            ```
        3. quicklist（redis7）内部是listpack
            * 与redis6版本的quicklist没有区别，只是将ziplist换成了listpack
4. set类型
    1. inset+hashtable
        * 通过 config get set* 命令读取配置
            ```shell
            1) "set-proc-title"
            2) "yes"
            3) "set-max-intset-entries"
            4) "512"
            ```
        * 集合元素都是long型，元素个数<=set-max-intset-entries时就是intset，反之就是hashtable
5. zset类型
    1. ziplist + skiplist （redis6）
        * 通过命令 config get zset* 获取配置
            ```shell
            1) "zset-max-ziplist-entries"
            2) "128"
            3) "zset-max-ziplist-value"
            4) "64"
            ```
        * 在满足上面的配置时使用ziplist，不满足时则使用skiplist
    2. listpack + skiplist （redis7）
        * 和redis6版本类似
## skiplist跳表
1. 为什么要做跳表？
    * 链表遍历的时候，时间复杂度最差会出现O(N)，我们优化一下，尝试空间换时间，给链表加个索引，称为“索引升级”，两两取首即可
2. 什么是跳表？
    * 借鉴数据库索引的思想
3. 跳表时间+复杂度介绍
    * 时间复杂度O(logN)
    * 空间复杂度O(N)



