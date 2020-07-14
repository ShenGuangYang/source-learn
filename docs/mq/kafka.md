# kafka 原理

## topic、partition 存储

### topic

 topic 是一个存储消息的逻辑概念，每一条消息都有一个 topic。物理上来说，不同 topic 的消息是分开存储的。

每个 topic 可以有多个生产者向他消费，也可以有多个消费者去消费其中的消息。

### Partition

每个 topic 可以划分多个分区，同一个 topic 下的不同分区包含的消息是不同的。每个消息在被添加到分区时，都会被分配一个 offset （偏移量），它是消息在此分区中的唯一编号，kafka 通过 offset 保证消息在分区内的顺序，所以kafka只能保证在同一个分区内的消息是有序的。



## 消息分发

消息是kafka中最基本的数据单元，由key、value两部分组成。在发送消息的时候可以指定key，那么producer会根据key和partition机制来判断当前这条消息应该发送并存储到哪个partition中。

**代码演示可以查看kafka基本应用**

### 消息默认的分发机制

默认情况下，kafka采用的是hash取模的分区算法。如果key为null，则会随机分配一个分区。

## 消息消费

![](image/kafka-1.png ':size=50%')

### 消费策略

如上图，如果一个topic有三个partition，那么不同数量consumer是如何进行消费的呢？

- 1个consumer与3个partition，那1个consumer消费所有partition中的消息
- 2个consumer与3个partition，那consumer_1 消费 p0，consumer_2 消费p1、p2
- 3个consumer与3个partition，那3个consumer分别消费2个partition中的消息
- 4个consumer与3个partition，那3个consumer分别消费2个partition中的消息，consumer_4不消费



那么什么时候回触发消费策略呢？

- 同一个consumer group 新增了消费者
- group中的消费者停机或者宕机
- topic新增了分区

### 分区分配策略

#### 范围分区（RangeAssignor）

对同一个topic里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序，算法如下

> 假设 n = 分区数 / 消费者数量
>
> m = 分区数 % 消费者数量
>
> 那么前m个消费者每个分配n+1个分区，后面的（消费者数量-m）个消费者每个分配n个分区

#### 轮循分区（RoundRobinAssignor）

轮循分区策略是把所有的partition和所有的consumer线程都列出来，然后按照hashcode进行排序。最后会通过轮循算法分配partition给consumer消费。



#### 粘滞策略（StickyAssignor）

在分区分配之后，会保留一份上一次的分配的副本。要是其中一个consumer宕机之后，会把宕机的分区进行重新分配，而未宕机的consumer会在原来持有分区上增加宕机消费的分区。

它主要有两个目的：

- 分区的分配尽可能均匀
- 分区的分配尽可能和上次保持相同



例子：

假设有4个topic(t1,t2,t3,t4)各有两个分区(p1,p2)，有三个消费者(c1,c2,c3)

那么最终分配后的结果如下

c1: t1p1,t2p2,t4p1

c2: t1p2,t3p1,t4p2

c3: t2p1,t3p2

消费过程中c1 宕机，导致重新分配，c2,c3的原有分配不变,轮循c1分给c2,c3

c2: t1p2,t3p1,t4p2,t2p2

c3: t2p1,t3p2,t1p1,t4p1

### rebalance

kafka提供了一个角色Coordinator 来执行对consumer group的管理。当consumer启动会跟coordinator进行通信。



### 如何确定coordinator？

消费者向kafka集群中的任一broke发送GroupCoordinatorRequest请求，服务器会返回一个负载最小的broke id，并将该节点设置成coordinator。

### join group的过程

主要分为两个过程

- jion: 表示加入到consumer group中，在这一步中，所有的成员都会向coordinator发送joinGroup的请 求。一旦所有成员都发送了joinGroup请求，那么coordinator会选择一个consumer担任leader角色， 并把组成员信息和订阅信息发送消费者

- sync: 完成分区分配之后，就进入了Synchronizing Group State阶段，主要逻辑是向GroupCoordinator发送 SyncGroupRequest请求，并且处理SyncGroupResponse响应，简单来说，就是leader将消费者对应 的partition分配方案同步给consumer group 中的所有consumer

## 分区副本机制

kafka的每个topic可以分为多个partition，并且多个partition会均匀分布在集群的各个节点下。虽然这种方式能够有效的对数据进行分片，但是对于每个partition来说都是单点的，当其中某个partition不可用时，那么这部分数据就没办法消费。所以kafka为了提高partition的高可用性提供了副本的概念（replica），通过副本机制来实现冗余备份。

每个分区可以有多个副本，并且在副本集合中会存在一个leader副本，所有读写请求都是由leader副本来进行处理的。剩余的其他副本都作为follower副本，follower副本会从leader副本同步消息日志。

### 副本的leader选举

kafka提供了书本复制算法保证，如果leader副本所在的broker节点宕机或者出现故障，或者分区的leader节点发生故障，这个时候怎么处理呢？

那么kafka必要保证从follower中选择一个新的leader副本。

kafka分区下有可能有很多副本用于实现冗余，从而进一步实现高可用。

副本根据角色的不同可以分为3类：

- leader：响应客户端的读写请求
- follower：被动备份leader副本中的数据
- ISR(in-sync replicas)：包含leader副本和所有leader副本保持同步的follewer副本
- LEO(log end offset):记录该副本底层日志中吓一跳消息的位移值
- HW(Hight Water)：已经同步的位移值，小于等于HW的所有消息都被认为是已同步的。

选举过程

1. kafkaController会监听zk的/brokers/ids节点路径，一旦发现有broker宕机，执行下面逻辑。

2. leader 副本在该broker上的分区就要重新进行leader选举，目前的选举策略是

   1. 优先从ISR列表中选出第一个作为leader副本
   2. 如果ISR为空，则查看该topic的unclean.leader.election.enable配置进行选举。

   

unclean.leader.election.enable

- true 代表允许选用非ISR列表的副本作为leader，那么就意味着数据可能丢失。
- false 代表不允许，直接抛出异常，造成leader副本选举失败



### 副本协同机制

消息的读写操作都只会有leader节点来接受和处理。follower副本只负责同步数据以及当leader副本所有broker挂了以后，会从follower副本中选取新的leader。

写请求首先由leader副本处理，之后follower副本会从leader上拉取写入的消息，这个过程会有一定延迟，导致follower副本中保存的消息略少于leader副本，但是只要没超出阈值都可以容忍。但是如果follower副本出现异常，比如宕机、停机、网络异常等原因没有同步到消息，那这个时候，leader就会把它踢出去。kafka通过ISR集合来维护分区副本信息。

### ISR

ISR表示目前“可用且消息量与leader相差不多的副本集合，这是整个副本集合的一个子集”。怎么去理解 可用和相差不多这两个词呢?具体来说，ISR集合中的副本必须满足两个条件

1. 副本所在节点必须维持着与zookeeper的连接
2. 副本最后一条消息的offset与leader副本的最后一条消息的offset之间的差值不能超过指定的阈值 (replica.lag.time.max.ms) replica.lag.time.max.ms:如果该follower在此时间间隔内一直没有追 上过leader的所有消息，则该follower就会被剔除ISR列表
3. ISR数据保存在Zookeeper的 /brokers/topics/<topic>/partitions/<partitionId>/state 节点中

follower副本把leader副本LEO之前的日志全部同步完成时，则认为follower副本已经追赶上了leader 副本，这个时候会更新这个副本的lastCaughtUpTimeMs标识，kafk副本管理器会启动一个副本过期检 查的定时任务，这个任务会定期检查当前时间与副本的lastCaughtUpTimeMs的差值是否大于参数 replica.lag.time.max.ms 的值，如果大于，则会把这个副本踢出ISR集合

### 如何处理所有的副本不工作的情况

在ISR中至少有一个follower时，kafka可以确保已经commit的数据不丢失，但是如果某个partition的所有replica都宕机了，就无法保证数据不丢失

1. 等待ISR中的任一个replica恢复，并且选它作为leader
2. 选择第一个恢复的replica作为leader

这就需要在可用性和一致性当中做出一个选择。

如果一定要等待ISR中的replica恢复，那不可用的时间就可能会相对较长。而且如果ISR中所有Replica都无法恢复，或者数据都丢失了，这个partition将永远不可用。

选择第一个恢复的replica作为leader，而这个replica不是ISR中的replica，那即使它不保证已经包含了所有已conmit的消息，它也会成为leader而作为consumer的数据源。



### 副本数据同步原理

了解了副本的协同过程以后，还有一个最重要的机制，就是数据的同步过程。他需要解决

1. 怎么传播消息
2. 在想消息发送端返回ack之前需要保证多少个replica已经接受到这个消息



producer 在发布消息到某个partition时，

- 先通过zk找到该partition的leader，然后发送到该partition的leader
- leader会将该消息写入其本地log。每个follower都从leader pull数据。这种方式上，follower存储的数据顺序与leader保持一致
- follower在收到该消息并写入其log后，想leader发送ack
- 一旦leader收到了ISR中所有replica的ack之后，该消息就被认为commit了，leader将增加HW并向producer发送ack

## 消息的存储

消费发送端发送消息到broker上以后，消息是如何持久化的呢？

首先我们需要了解的是，kafka是使用日志文件的方式来保存生产者和发送者的消息，每条消息都有一 个offset值来表示它在分区中的偏移量。Kafka中存储的一般都是海量的消息数据，为了避免日志文件过 大，Log并不是直接对应在一个磁盘上的日志文件，而是对应磁盘上的一个目录，这个目录的命名规则 是<topic_name>_<partition_id>



### 日志文件的清除策略以及压缩策略

#### 日志清除策略

日志是分段存储，一方面能够减少单个文件内容的大小，另一方面，方便kafka进行日志清理。日志清理策略主要有两个

1. 根据消息的保留时间，当消息在kafka中保存的时间超过了指定的时间，就会触发清理过程
2. 根据topic存储的数据大小，当topic所占的文件大小大于一点的阈值，则可以开始删除最旧的消息

通过log.retention.bytes和log.retention.hours这两个参数来设置，当其中任意一个达到要求，都会执行删除。



#### 日志压缩策略

kafka还提供了日志压缩的功能，通过这个功能可以有效的减少日志文件的大小。在实际场景中，消息的k-v之间的对应关系是不断变化的，但是消费者只关心最新的值。因此，日志压缩功能就是后台启动cleaner线程池，定期将相同的可以进行合并，只保留最新的k-v值。



## 磁盘存储的性能问题

### 顺序读写

kafka采用顺序读写的方式存储数据，优化了磁盘读写的性能问题（减少寻址的过程）

### 零拷贝

消息从发送到落地保存，broker维护的消息日志本身就是文件目录，每个文件都是二进制保存，生产者和消费者使用相同的格式来处理。在消费者获取消息时，服务器先从硬盘读取数据到内存，然后把内存中的数据通过socker发送给消费者。操作系统实现步骤：

- 操作系统将数据从磁盘读入到内核空间
- 应用程序将数据从内核空间读入到用户空间缓存
- 应用程序将数据写回到内核空间到socker缓存中
- 操作系统将数据从socker缓冲区复制到网卡缓冲区

通过零拷贝技术减少这些复杂的数据复制操作，也会减少上下文切换的次数。

java使用的api：FileChannel.transferTo

### 页缓存

页缓存是操作系统实现的一种主要的磁盘缓存，但凡设计到缓存的，基本都是为了提升i/o性能，所以页 缓存是用来减少磁盘I/O操作的。

磁盘高速缓存有两个重要因素:

- 第一，访问磁盘的速度要远低于访问内存的速度，若从处理器L1和L2高速缓存访问则速度更快。 
- 第二，数据一旦被访问，就很有可能短时间内再次访问。正是由于基于访问内存比磁盘快的多，所 以磁盘的内存缓存将给系统存储性能带来质的飞越。

当一个进程准备读取磁盘上的文件内容时， 操作系统会先查看待读取的数据所在的页(page)是否在页 缓存(pagecache)中，如果存在(命中)则直接返回数据， 从而避免了对物理磁盘的I/0操作;如果没有 命中， 则操作系统会向磁盘发起读取请求并将读取的数据页存入页缓存， 之后再将数据返回给进程。 同样，如果 一 个进程需要将数据写入磁盘， 那么操作系统也会检测数据对应的页是否在页缓存中，如 果不存在， 则会先在页缓存中添加相应的页， 最后将数据写入对应的页。 被修改过后的页也就变成了 脏页， 操作系统会在合适的时间把脏页中的数据写入磁盘， 以保持数据的一致性

## kafka消息的可靠性

没有一个中间件能够做到百分百的完全可靠。kafka是如何实现最大可能的可靠性？

- 分区副本，创建更多的分区来提升可靠性，但是分区数过多也会带来性能上的开销
- acks，生产者发送消息的可靠性，也就是要保证我这个消息一定是到了broker并且完成了多副本的持久化。
- 保证消息到了broker之后，消费者也需要保证消息被消费了，消费者消费消息是默认自动批量提交了。这有一个时间窗口，会带来重复消费或者消息丢失的情况，所以高可靠性要求的程序可以使用手动提交。

