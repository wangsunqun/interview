## 基础知识

![](../resources/redis.jpg)

- **redis和memcache区别**
    - redis是串行的，一条数据处理完下一条才能处理
    - memcache并行多线程执行。
- **redis的key可以用冒号来分割**，可以自动归档，在可视化界面下就可以看到所有key都在同一个文件夹下。
- **aof和rdb区别**
    - rdb就是快照，相当于把整个数据库备份一遍，如果数据文件特别大，可能导致 Redis 短暂无法提供服务
    - aof除了查以为外的命令都会被记录下来，每隔 1 秒存储 Redis 命令到日志文件，日志文件以 append-only 模式写入
    - 多数使用 AOF+RDB 方式来持久化，用 AOF 来保证数据不丢失，作为数据恢复的第一选择; 用 RDB 来做不同程度的冷备，AOF 文件都丢失或损坏时可以使用 RDB 恢复数据
- **过期策略（定期删除+惰性删除）**
    - 定期删除：指的是 Redis 默认是每隔 100ms 就随机抽取一些设置了过期时间的 key，检查其是否过期，如果过期就删除
    - 惰性删除：在你获取某个 key 的时候，Redis 会检查一下 ，这个 key 如果设置了过期时间那么是否过期了？如果过期了此时就会删除，不会给你返回任何值
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

## 架构

1. **单机**  
   就是单节点，读写都在一个节点上
2. **主从**  
   只有主提供服务，从只是做备份。但是可以通过配置实现读写分离
3. **主从 + keepalived**  
   springboot连接keepalived的VIP，当主节点挂了，飘移到从节点。当主节点回复了，由于keepalived权重关系，又会重新飘回主节点
4. **主从 + sentinel（经典的一主两从三哨兵模式）**  
   springboot连接sentinel，由sentinel转发到redis集群，sentinel作用有点像keepalived，当主节点挂了，会自动选举一个主节点。和keepalived一样当主挂了才会选举新主
   ![](../resources/redis16.jpg)
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
   从redis3.0之后版本支持redis-cluster集群，Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。主之间平均分配hash槽(一共16384)
   ，数据不同步。从节点同步主节点内容。但是取数据是根据crc16(key) 取模来获取槽id，再根据id到对于的节点取值。 一个哈希槽可以存储多个key-value。数据就是存储在哈希槽里。

    - 优点：
        1. 无中心架构（不存在哪个节点影响性能瓶颈），少了 proxy层。
        2. 数据按照 slot 存储分布在多个节点，节点间数据共享，可动态调整数据分布。
        3. 可扩展性，可线性扩展到 1000 个节点，节点可动态添加或删除。
        4. 高可用性，部分节点不可用时，集群仍可用。通过增加 Slave做备份数据副本
        5. 实现故障自动 failover，节点之间通过 gossip 协议交换状态信息，用投票机制完成 Slave到 Master的角色提升。
    - 缺点：
        1. 资源隔离性较差，容易出现相互影响的情况。
        2. 数据通过异步复制,不保证数据的强一致性
    - <font color=red>**选举**</font>
      当slave发现自己的master变为FAIL状态时，便尝试进行Failover，以期成为新的master。由于挂掉的master可能会有多个slave，从而存在多个slave竞争成为master节点的过程  
      ![](../resources/redis13.jpg)
      其过程如下：
        1. slave发现自己的master变为FAIL
        2. 将自己记录的集群currentEpoch加1，并广播FAILOVER_AUTH_REQUEST信息
        3. 其他节点收到该信息，只有master响应，判断请求者的合法性，并发送FAILOVER_AUTH_ACK，对每一个epoch（比如都是11，那么一个master只能响应epoch=11 1次）只发送一次ack
        4. 尝试failover的slave收集FAILOVER_AUTH_ACK
        5. 超过半数后变成新Master
        6. 广播Pong通知其他集群节点。

    - <font color=red>**创建集群的过程**</font>
      ![](../resources/redis14.jpg)

## 主从同步

**以下是第一次启动时候的过程，为全量同步，后面就会是增量同步**  
<font color=red>**全量同步**</font>  
2.8以前（旧版方式）

1. 从服务器连接主服务器，发送 SYNC 命令（全量复制）；
2. 主服务器接收到 SYNC 命名后，开始执行 BGSAVE 命令生成 RDB 文件并使用缓冲区记录此后执行的所有写命令；
3. 主服务器 BGSAVE 执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令；
4. 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照；
5. 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令；
6. 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；

全量同步图：  
![](../resources/redis17.jpg)  
2.8以后（新方式）  
使用PSYNC（部分复制）代替SYNC上述的方案有个缺陷，就是SYNC在从断线重连(不是重启，可能只是网络不好断开了)后，还是会把完整的rdb复制一份然后执行PSYNC可以只复制断线这段时间的数据

<font color=red>**增量同步**</font>  
Redis增量复制是指Slave初始化后开始正常工作时主服务器发生的写操作同步到从服务器的过程。  
增量复制的过程主要是主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令。

## Raft

**redis的sentinel模式下有使用，用来选主**  
<font color=red>**简明扼要：**</font>
例如redis的sentinel一开始都是follower节点，当监控的master节点一定时间没有返回，那么监听这个master的这些sentinel节点就变成candidate节点，并开始发选票，哪个选票超过半数就当leader，然后对slaver节点升级

#### 节点的状态

每个节点有三个状态，他们会在这三个状态之间进行变换。客户端只能从主节点写数据，从节点里读数据。  
![](../resources/raft.jpg)

#### 选主流程

初始是Follwer状态节点，等100-300MS没有收到LEADER节点的心跳就变候选人。候选人给大家发选票，候选人获得大多数节点的选票就变成了LEADER节点。  
![](../resources/raft1.jpg)

#### 日志复制流程

每次改变数据先记录日志，日志未提交不能改节点的数值。然后LEADER会复制数据给其他follower节点，并等大多数节点写日志成功再提交数据。  
![](../resources/raft2.jpg)

#### 选举超时

每个节点随机等150到300MS，如果时间到了就开始发起选票，因为有的节点等的时间短，所以它会先发选票，从而当选成主节点。但是如果两个候选人获得的票一样多，它们之间就要打加时赛，这个时候又会重新随机等150到300MS，然后发选票，直到获得最多票当选成主节点。  
![](../resources/raft3.jpg)

#### 心跳超时

每个节点会记录主节点是谁，并且和主节点之间维持一个心跳超时时间，如果没有收到主节点回复，从节点就要重新选举候选人节点。  
![](../resources/raft4.jpg)

#### 集群中断

当集群之间的部分节点失去通讯时，主节点的日志不能复制给多个从节点就不能进行提交。  
![](../resources/raft5.jpg)

#### 集群恢复

当集群恢复之后，原来的主节点发现自己不是选票最多的节点，就会变成从节点，并回滚自己的日志，最后主节点会同步日志给从节点，保持主从数据的一致性。   
![](../resources/raft6.jpg)

## Gossip（八卦）协议

**redis的cluster模式下有使用，用来让集群中的每个实例都知道其他所有实例的状态信息**  
<font color=red>**简明扼要：**</font> 每隔一段时间，每个节点都会随机选择几个节点发送Gossip消息，其他节点会再次随机选择其他几个节点接力发送消息。这样一段时间过后，整个集群都能收到这条消息  
Redis 集群是去中心化的，彼此之间状态（ 元数据、故障检测、配置更新、故障转移授权）同步靠 gossip 协议通信，集群的消息有以下几种类型：

1. Meet 通过「cluster meet ip port」命令，已在集群的节点会向新的节点发送邀请，加入现有集群。
2. Ping 节点每秒会向集群中其他节点发送 ping 消息，消息中带有自己已知的两个节点的地址、槽、状态信息、最后一次通信时间等。
3. Pong 节点收到 ping 消息后会回复 pong 消息，消息中同样带有自己已知的两个节点信息。
4. Fail 节点 ping 不通某节点后，会向集群所有节点广播该节点挂掉的消息。其他节点收到消息后标记已下线。

由于去中心化和通信机制，Redis Cluster 选择了最终一致性和基本可用。

## 缓存一致性方案

redis和mysql数据一致性方案，首先你要明白这问题是无解，只要你把每种方案的利弊说清楚即可

- 先更新缓存，再更新数据库
    - 问题：
        - 缓存更新成功，数据库失败然后回滚了
        - 线程并发。线程A(更新),B(更新)，A更新了缓存，B更新了缓存，B更新了数据库，A更新了数据库
        - 业务问题。如果是写多读少的业务，那么有很多次无效更新
- 先更新数据库，再更新缓存
    - 问题：
        - 更新数据库成功，缓存更新失败
        - 线程并发。线程A(更新),B(更新)，A更新了数据库，B更新了数据库，B更新了缓存，A更新了缓存
        - 业务问题。如果是写多读少的业务，那么有很多次无效更新
- 先删缓存，再更新数据库
    - 问题：
        - 线程并发。线程A(更新),B(查询)，A删除了缓存，B读取数据库并写到缓存，A更新数据库
- 先更新数据库，再删缓存
    - 问题：
        - 更新数据库成功，删除缓存失败
        - 线程并发。线程A(查询),B(更新)，缓存刚好过期了，A去库里查了，B更新了数据库，B删除缓存，A将旧值写回

**总结：**  
目前网上比较认同最后一种，但是也要注意2个点，如果读请求很多，删除缓存会不会造成击穿。另一个针对最后一种的第一个问题可以用@Transaction或者订阅binlog+消息队列来解决。

## redis挂了怎么办

又是一个扯淡的面试题。遇到这个问题，面试官心里一般有2种：1、考验你高可用思想。2、他自己背了4个9的绩效来问方案的 可以按照如下回答，注意该问题可以作为模板，套用kafka挂了，mysql挂了怎么办：

1. 事前：要解决问题，必须要知道原因，redis为什么会挂?
    - 运维：是否做了高可用，自动扩缩容，异地多活，监控，警报
    - 测试：是否有做合理的压测
    - 产品、运营、PM：是否有对请求量，新增用户有合理评估
2. 事后：也就是真的发生了怎么办？
    - 降级：快速失败、用消息队列暂存等服务恢复了再消费、用本地内存（也要看业务）

**总结：**  
这类型的问题千万不要上来就干花里胡哨的解决方法，主要是想知道你有没有高可用架构思维。先把事前的吹一波，事后的再搞点高级词汇，熔断降级，暂存后补等等基本完事。

## 线程模型

<font color=red>**redis6以下**</font>  
是单线程的reactor模型，网络io和计算都是用同一个线程执行的  
![](../resources/redis9.jpg)

<font color=red>**redis6**</font>  
改为io是多线程，但是工作线程还是只有一个
![](../resources/redis10.jpg)

## redis线程模型的误区

（以下内容为直接复制网上一个人的，具体是哪的我也忘了）

### 问题：为什么Redis使用单线程处理请求，还要使用队列，出队列跟入队列都是同一个线程，这操作不是多余吗？

我这里指的是Redis的线程模型，既然是单线程使用IO复用技术处理并发请求，那为什么接收到请求事件后还要将事件放到队列里面去，直接使用当前线程处理事件不是省去了入队跟出队的消耗吗？  
![](../resources/redis15.jpg)

首先强调一句：  
**网上大部分描绘Redis事件模型或者Reactor网络模式的图，都存在误导！极其容易将简单的问题复杂化。**  
什么**文件事件分配器**，其实就是for循环。什么**连接应答处理器/命令请求处理器/命令回复处理器**，都只过是不同的回调函数（从Redis事件库AE的角度上看，只不过是之前注册的函数指针被执行了而已）。  
看到名字叫”**器**“是不是感觉高大上。但很容易把人整懵逼。  
回说这个队列，这里的队列并不是Redis代码用使用了什么队列存储事件，指的是epoll的内部实现的数据结构。Redis自己没有显式引入队列（Redis
6.0之前，单线程模式）。所谓的队列msg3、msg2、msg1不过是epoll_wait()函数的出参。看下源码：

```
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
	aeApiState *state = eventLoop->apidata;
	int retval, numevents = 0;

	retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
	tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
	if (retval > 0) {
		int j;

		numevents = retval;
		for (j = 0; j < numevents; j++) {
			int mask = 0;
			struct epoll_event *e = state->events+j;

			if (e->events & EPOLLIN) mask |= AE_READABLE;
			if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
			if (e->events & EPOLLERR) mask |= AE_WRITABLE|AE_READABLE;
			if (e->events & EPOLLHUP) mask |= AE_WRITABLE|AE_READABLE;
			eventLoop->fired[j].fd = e->data.fd;
			eventLoop->fired[j].mask = mask;
		}
	}
	return numevents;
}
```  

题主图中所说的队列，可能指的是源码中的state->events。这个其实就是 struct epoll_event *
类型。你要用epoll都需要这个，并不是说Redis把IO多路复用（epoll）触发的事件又存到了某个自己的队列中！这个就是epoll自己接口的出参而已……