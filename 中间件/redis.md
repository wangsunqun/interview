## 基础知识

![](../resources/redis.jpg)

- **redis和memcache区别**
    - redis是串行的，一条数据处理完下一条才能处理
    - memcache并行多线程执行。
- **redis的key可以用冒号来分割**，可以自动归档，在可视化界面下就可以看到所有key都在同一个文件夹下。
- **aof和rdb区别**
    - rdb就是快照，相当于把整个数据库备份一遍
    - aof除了查以为外的命令都会被记录下来
- **淘汰算法**
    - volatile-lru：使用LRU算法进行数据淘汰（淘汰上次使用时间最早的，且使用次数最少的key），只淘汰设定了有效期的key
    - allkeys-lru：使用LRU算法进行数据淘汰，所有的key都可以被淘汰
    - volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键
    - allkeys-lfu：从所有键中驱逐使用频率最少的键
    - volatile-random：随机淘汰数据，只淘汰设定了有效期的key
    - allkeys-random：随机淘汰数据，所有的key都可以被淘汰
    - volatile-ttl：淘汰剩余有效期最短的key
    - no-enviction：不淘汰（默认）
- **查询慢日志（slowlog get）**
    - ![](../resources/redis8.jpg)

## [【redis 底层数据结构与数据类型 ppt】](../resources/redis.pptx)

### 底层数据结构

1. 简单动态字符串（SDS）
   ![](../resources/redis1.jpg)

2. 链表 就是简单的双向链表
   ![](../resources/redis2.jpg)

3. 哈希表 跟hashmap原理差不多 扩容：
    - 有两个哈希表h0和h1，先扩大h1大小
    - murmurhash(key) & sizemark
    - 渐近式哈希，复制的过程不是一口气，完成的 而是一部分一部分完成的

4. 跳跃表（zrange、zrank、zrangebyscore） 每个节点有个level数组，level具体是多少是随机生成的。level每一个元素里有指向下一个节点的指针和跨越多少个节点，用来计算rank
   redis对跳跃表做了改良：①使得第一层是一个双向链表（ 这是为了方便以倒序方式获取一个范围内的元素）。②每个节点存了到每个下节点跨越的个数（跟指针是一一对应，用来计算rank）
   详细看： https://www.cnblogs.com/Elliott-Su-Faith-change-our-life/p/7545940.html
   ![](../resources/redis3.jpg)
   ![](../resources/redis4.jpg)


5. 整数集合  
   ![](../resources/redis5.jpg)

6. 压缩列表（ziplist） 一个内存空间连续的双向链表（普通链表内存不连续）  
   5种redis数据结构都是由2种底层结构组成，根据不同情况使用不同数据结构，但是基本都是量少才用ziplist  
   ![](../resources/redis6.jpg)

### 数据类型

#### 基础数据类型

kv、list、map、set、zset

#### 高级数据类型

- **bitmap**
    - 方法
        - getbit key offset: 对key所存储的字符串值，获取指定偏移量上的位（bit）
        - setbit key offset value: 对key所存储的字符串值，设置或清除指定偏移量上的位（bit）。返回值为该位在setbit之前的值
        - bitcount key [start end]: 获取位图指定范围中位值为1的个数，如果不指定start与end，则取所有
        - bitop op destKey key1 [key2...]: 做多个BitMap的and（交集）、or（并集）、not（非）、xor（异或）操作并将结果保存在destKey中
        - bitpos key tartgetBit [start end]: 计算位图指定范围第一个偏移量对应的的值等于targetBit的位置。找不到返回-1。start与end没有设置，则取全部
    - 使用场景：参考地址：https://www.cnblogs.com/wuhaidong/articles/10389484.html
        - 用户签到，在线，统计活跃用户
        - 布隆过滤器
    -
  offset优化建议：由于最大大小为512M，那么我们如果要把一个很大的long（比如用户id）当做offset，一来浪费空间，二来也存不下。可以采用如下优化方法：拆分n个key，例如：bitmap的key是userId/n，offset可以设置为userId%n

- **hyperloglog**
    - 方法
        - pfadd：增加计数（和set的sadd用法一样，来一个往里面放一个）
        - pfcount：获取计数（和scard的用法一样，直接获取计数）
        - pfmerage：将多个pf计数累加再一起形成一个新的pf值
    - 使用场景：用于做基数统计。什么是基数? 比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。
      基数估计就是在误差可接受的范围内，快速计算基数

## 穿透、击穿、雪崩

1. 穿透  
   在高并发下，查询一个不存在的值时（黑客行为），缓存不会被命中，导致大量请求直接落到数据库上，如活动系统里面查询一个不存在的活动。
    - 布隆过滤器
    - 缓存空值，但是时间不能太长，下次进来是直接返回不存在，但是这种情况无法过滤掉动态的key，就是说每次请求进来都是不同的key，这样还是会造成这个问题
2. 击穿  
   在高并发下，对一个特定的值进行查询，但是这个时候缓存正好过期了，缓存没有命中，导致大量请求直接落到数据库上，如活动系统里面查询活动信息，但是在活动进行过程中活动缓存突然过期了。  
   通过synchronized+双重检查机制：某个key只让一个线程查询，阻塞其它线程

```
private static volaite Object lockHelp=new Object();

   public String getValue(String key){
     String value=redis.get(key,String.class);
     
     if(value=="null"||value==null||StringUtils.isBlank(value){
         synchronized(lockHelp){
                value=redis.get(key,String.class);
                 if(value=="null"||value==null||StringUtils.isBlank(value){
                     value=db.query(key);
                      redis.set(key,value,1000);
                  }
            }
           }    

        return value;
   }
```

3. 雪崩  
   在高并发下，大量的缓存key在同一时间失效，导致大量的请求落到数据库上，如活动系统里面同时进行着非常多的活动，但是在某个时间点所有的活动缓存全部过期。
    - 可以通过缓存reload机制，预先去更新缓存，再即将发生大并发访问前手动触发加载缓存
    - 不同的key，设置不同的过期时间，具体值可以根据业务决定，让缓存失效的时间点尽量均匀

## 线程模型

<font color=red>**redis6以下**</font>  
是单线程的reactor模型，网络io和计算都是用同一个线程执行的  
![](../resources/redis9.jpg)

<font color=red>**redis6**</font>  
改为io是多线程，但是工作线程还是只有一个
![](../resources/redis10.jpg)

## 架构

1. **单机**  
   就是单节点，读写都在一个节点上
2. **主从**  
   只有主提供服务，从只是做备份。但是可以通过配置实现读写分离
3. **主从 + keepalived**  
   springboot连接keepalived的VIP，当主节点挂了，飘移到从节点。当主节点回复了，由于keepalived权重关系，又会重新飘回主节点
4. **主从 + sentinel（经典的一主两从三哨兵模式）**  
   springboot连接sentinel，由sentinel转发到redis集群，sentinel作用有点像keepalived，当主节点挂了，会自动选举一个主节点。和keepalived一样当主挂了才会选举新主
5. **2个twemProxy + 5个redis主 + keepalived**  
   ![](../resources/redis11.jpg)  
   Twemproxy 是一个 Twitter 开源的一个 redis 和 memcache 快速/轻量级代理服务器； Twemproxy 是一个快速的单线程代理程序，支持 Memcached ASCII 协议和 redis 协议。
    - 优点：
        1. 多种 hash 算法：MD5、CRC16、CRC32、CRC32a、hsieh、murmur、Jenkins
        2. 支持失败节点自动删除
        3. 后端 Sharding 分片逻辑对业务透明，业务方的读写方式和操作单个 Redis 一致
    - 缺点：
        1. 增加了新的 proxy，需要维护其高可用。
        2. failover逻辑需要自己实现，其本身不能支持故障的自动转移可扩展性差，进行扩缩容都需要手动干预
6. **culster**   
   springboot要把主从全写上去，网上说jedis才可以做到主从切换，template不行。有待验证  
   ![](../resources/redis12.jpg)  
   从redis3.0之后版本支持redis-cluster集群，Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。

    - 优点：
        1. 无中心架构（不存在哪个节点影响性能瓶颈），少了 proxy层。
        2. 数据按照 slot 存储分布在多个节点，节点间数据共享，可动态调整数据分布。
        3. 可扩展性，可线性扩展到 1000 个节点，节点可动态添加或删除。
        4. 高可用性，部分节点不可用时，集群仍可用。通过增加 Slave做备份数据副本
        5. 实现故障自动 failover，节点之间通过 gossip 协议交换状态信息，用投票机制完成 Slave到 Master的角色提升。
    - 缺点：
        1. 资源隔离性较差，容易出现相互影响的情况。
        2. 数据通过异步复制,不保证数据的强一致性
    - <font color=red>**选举**</font> 当slave发现自己的master变为FAIL状态时，便尝试进行Failover，以期成为新的master。由于挂掉的master可能会有多个slave，从而存在多个slave竞争成为master节点的过程  
      ![](../resources/redis13.jpg)
      其过程如下：
        1. slave发现自己的master变为FAIL
        2. 将自己记录的集群currentEpoch加1，并广播FAILOVER_AUTH_REQUEST信息
        3. 其他节点收到该信息，只有master响应，判断请求者的合法性，并发送FAILOVER_AUTH_ACK，对每一个epoch（比如都是11，那么一个master只能响应epoch=11 1次）只发送一次ack
        4. 尝试failover的slave收集FAILOVER_AUTH_ACK
        5. 超过半数后变成新Master
        6. 广播Pong通知其他集群节点。

一、主之间平均分配hash槽，数据不同步。从节点同步主节点内容。但是取数据是根据crc16(key) 取模来获取槽id，再根据id到对于的节点取值。
一个哈希槽可以存储多个key-value。数据就是存储在哈希槽里。五、创建集群的过程Gossip（八卦）协议的使用Redis 集群是去中心化的，彼此之间状态（ 元数据、故障检测、配置更新、故障转移授权）同步靠 gossip
协议通信，集群的消息有以下几种类型：

1. Meet 通过「cluster meet ip port」命令，已有集群的节点会向新的节点发送邀请，加入现有集群。
2. Ping 节点每秒会向集群中其他节点发送 ping 消息，消息中带有自己已知的两个节点的地址、槽、状态信息、最后一次通信时间等。
3. Pong 节点收到 ping 消息后会回复 pong 消息，消息中同样带有自己已知的两个节点信息。
4. Fail 节点 ping 不通某节点后，会向集群所有节点广播该节点挂掉的消息。其他节点收到消息后标记已下线。

由于去中心化和通信机制，Redis Cluster 选择了最终一致性和基本可用。
