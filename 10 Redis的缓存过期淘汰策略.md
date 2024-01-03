## Redis内存满了怎么办？
1. redis默认内存多少？在哪里查看？如何修改？
    1. 查看Redis最大占用内存
        * conf文件--> maxmemory属性
            > maxmemory 256mb
        * 命令查看
            > config get maxmemory
    2. 默认内存多少可以用？
        * 如果不设置内存大小或者设置内存大小为0，在64位操作系统下不限制内存大小，在32位系统下最多使用3GB内存
    3. 一般生产环境配置多大内存？
        * 一般推荐Redis设置内存为最大物理内存的四分之三
    4. 如何修改redis内存设置？
        * 命令修改（临时修改）
            > config set maxmemory xxxx
        * 修改config文件（永久修改）
    5. 如何查看redis内存使用情况？
        * redis命令
            > info memory
2. Redis内存满了会怎么样？
    * 报错OOM
## 向redis中写的数据怎么没了？如何删除的？
1. redis过期键的删除策略
    * 如果redis中一个键过期了，是立即删除吗？----> 当然不是
2. 三种不同的删除策略
    1. 立即删除（时间换空间）
        * Redis不可能时时刻刻便利所有被设置了过期时间的key，来检测数据是否已经到达过期时间，然后对它进行删除
        * 立即删除可以保证内存中数据的最大新鲜度，因为它保证过期键值会在过期后马上被删除，其所占用的内存也会随之释放。但是立即删除对cpu是最不友好的。因为删除操作会占用cpu的时间，如果刚好碰上了cpu很忙的时候，比如正在做交集或排序等计算的时间，就会cpu造成额外的压力
        * 会产生大量的性能消耗，同时也会影响数据的读取操作
    2. 惰性删除（空间换时间）
        * 不去查看数据是否过期。等到读取数据时，如果数据已过期，返回不存在并删除
        * 缺点是对内存不友好  ----> 如果存在大量key过期后没有再次读取的情况，那么不就会存在大量的垃圾数据？
        * 如何开启？----> config文件中配置
            > lazyfree-lazy-eviction=yes
    3. 定期删除
        * 折中方案。redis每隔一段时间删除过期的key并通过限制删除操作执行时长和频率来减少对CPU时间的影响
        * 周期性轮询redis库中的时效性数据，采用随机抽取的策略，利用过期数据占比的方式控制删除频度
            1. 特点1：CPU性能占用设置有峰值，检测频率可自定义设置
            2. 特点2：内存压力不是很大，长期占用内存的冷数据会被持续清理
        * 这种策略的难点在于如何确定删除操作执行的时长和频率：如果频率太频繁或者执行的时间太长，就会退化成立即删除策略；如果频率太低下或者执行时间太短，就会退化成惰性删除策略
    4. 定期删除还是有漏洞，因为定期删除是定期抽查部分key进行删除，有可能存在某些key一直没有被抽查到，同时也没有被访问到
## Redis的缓存淘汰策略（重点）
1. 在哪配置？
    * config文件中的 maxmemory-policy 属性
2. LRU算法和LFU算法的区别？
    1. 名字不同
        * LRU means Least Recently Used
        * LFU means Least Frequently Used
    2. 针对对象不同
        * LRU淘汰最长时间未被使用的
        * LFU淘汰一定时间内被访问次数最少的
3. 内存淘汰策略有哪些？
    * volatile-lru
        > maxmemory-policy volatile-lru # 对所有设置了过期时间的key使用LRU算法
    * allkeys-lru
        > maxmemory-policy allkeys-lru  # 对所有key使用LRU算法
    * volatile-lfu
        > maxmemory-policy volatile-lfu  # 对所有设置了过期时间的key使用LFU算法
    * allkeys-lfu
        > maxmemory-policy allkeys-lfu   # 对所有key使用LFU算法
    * volatile-random
        > maxmemory-policy volatile-random # 对所有设置了过期时间的key随机删除
    * allkeys-random
        > maxmemory-policy allkeys-random  # 对所有key随机删除
    * volatile-ttl
        > maxmemory-policy volatile-ttl   # 删除马上要过期的key
    * noeviction
        > maxmemory-policy noeviction  # 不会淘汰任何key
4. 如何选择？
    * 在所有的key都是最近最经常使用，那么就需要选择allkeys-lru进行置换最近最不经常使用的key；如果不确定使用哪种策略，那么推荐使用allkeys-lru
    * 如果所有的key的访问概率都是差不多的，那么可以选用allkeys-random策略去置换数据
    * 如果对数据有足够的了解，能够为key指定kint（通过expire/ttl指定），那么可以选择volatile-ttl进行置换

