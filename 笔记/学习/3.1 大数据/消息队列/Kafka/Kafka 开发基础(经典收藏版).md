- [Kafka 开发基础(经典收藏版)_000X000的博客-CSDN博客](https://blog.csdn.net/ytp552200ytp/article/details/123177153)

### 为什么需要使用[kafka](https://so.csdn.net/so/search?q=kafka&spm=1001.2101.3001.7020)

从本质上来讲，是因为互联网发展太快，使用单体[架构](https://so.csdn.net/so/search?q=架构&spm=1001.2101.3001.7020)无疑会是的体量巨大。而微服务架构可以很好的解决这个问题，但是服务与服务之间会还是出现耦合、访问控制等问题。 消息队列可以很好的满足这些需要。它常用来实现：**异步处理、服务解耦、流量控制**



### 异步处理

随着业务的不断增加，通常会在原有的服务上添加上新服务，这样会出现**请求链路越来越长，链路latency也就逐步增加**。例如：最开始的电商项目，可能就是简简单单的扣库存、下单。慢慢地又加上了积分服务、短信服务等。链路增长不可避免的latency增加。

相对于扣库存、下单，积分和短信没必要恢复的那么及时。所以只需要在下单结束的时候结束那个流程，把消息传给[消息队列](https://so.csdn.net/so/search?q=消息队列&spm=1001.2101.3001.7020)中就可以直接返回响应了。而且短信服务和积分服务可以并行的消费这条消息。这样**响应的速度更快，用户体验更好**；**服务异步执行，系统整体latency（相对使用同步机制而言）也下降了**。



![img](https://img-blog.csdnimg.cn/cc0f42c9717d4746908274264fdb36ad.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)

异步处理

### 服务解耦

上面说的新加了短信服务和积分服务，现在又需要添加数据分析服务、以后可能又加一个策略服务等。可以发现订单的后续链路一直在增加，为了适配这些功能，就需要不断的修改订单服务，下游任何一个服务的接口改变都可能会影响到订单服务。

这个时候可以采用消息队列来解耦，订单服务只需要把消息塞到消息队列里面，下游服务谁要这个消息谁就订阅响应的topic。这样订单服务就不用被拿捏住了！！



![img](https://img-blog.csdnimg.cn/a27e8fd215b1407e8f61bb4ca86cfd1a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



服务解耦

### 流量治理

后端服务相对而言是比较脆弱的，因为业务较重，处理时间长。如果碰到高QPS情况，很容易顶不住。比如说题库数据写入到ES索引中，数据都是千万级别的。这个时候使用中间件来做一层缓冲，消息队列是个很不错的选择。

变更的消息先存放到消息队列中，后端服务尽自己最大的努力去消费队列中消费数据。

同时，对于一些不需要及时地响应处理，且业务处理逻辑复杂、流程长，那么数据放到消息队列中，消费者按照自己的消费节奏走，也是很不错的选择。

上述分别对应着 **生产者生产过快** 和 **消费者消费过慢** 两种情况，使用消息队列都能很好的起到缓冲作用。



![img](https://img-blog.csdnimg.cn/e941300a331a49b3a6df6aa78635614d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



流量治理

### 总结一下

 

kafka特点：

1. 解耦合。消息队列提供了借口，生产者和消费者能够独立的完成读操作和写操作。
2. 高吞吐率。即使是在廉价的商用机器上也能做到单机支持每秒100K条消息的传输
3. 信息传输快。以时间复杂度为`O（1）`的方式提供持久化能力，即使对`TB级`以上数据也能保证常数时间的访问性能
4. 可提供持久化。消息存储在中间件上，数据持久化，直到全部被处理完，通过这一方式规避了数据丢失的风险。





### kafka适用场景

根据上述功能和特点，kafka主要有以下使用场景：

1. 信息系统 `Messaging` 。 在这个领域中，kafka常常被拿来与传统的消息中间件进行对比，如RabbitMQ。
2. 网站活动追踪 `Website Activity Tracking`
3. 监控 `Metrics`
4. 日志收集 `Log Aggregation`
5. 流处理 `Stream Processing`
6. 事件溯源 `Event Sourcing`
7. 提交日志 `Commit Log`



具体可见：使用场景

### kafka组件



![img](https://img-blog.csdnimg.cn/fd66770167b3444387b0f6dab05c9ba7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



kafka架构图

1. **Producer** ： 消息生产者，就是向 Kafka发送数据 ；

2. **Consumer** ： 消息消费者，从 Kafka broker 取消息的客户端；

3. **Consumer Group （CG）**： 消费者组，由多个 consumer 组成。 消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。 所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。

4. **Broker** ：经纪人 一台 Kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker可以容纳多个 topic。

5. **Topic** ： 话题，可以理解为一个队列， 生产者和消费者面向的都是一个 topic；

6. **Partition**： 为了实现扩展性，一个非常大的 topic 可以分布到多个 broker（即服务器）上，一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列；如果一个topic中的partition有5个，那么topic的并发度为5.

7. **Replica**： 副本（Replication），为保证集群中的某个节点发生故障时， 该节点上的 partition 数据不丢失，且 Kafka仍然能够继续工作， Kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本，一个 leader 和若干个 follower。

8. **Leader**： 每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是 leader。

9. **Follower**： 每个分区多个副本中的“从”，实时从 leader 中同步数据，保持和 leader 数据的同步。 leader 发生故障时，某个 Follower 会成为新的 leader。

10. Offset

    

    ： 每个Consumer 消费的信息都会有自己的序号，我们称作当前队列的offset。即

    消费点位标识消费到的位置

    。每个消费组都会维护订阅的Topic 下每个队列的offset

    



![img](https://img-blog.csdnimg.cn/ffca33b8f79b4c98bb470e8d9dbc972e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



### 基本配置

参见官网配置

### kafka基本使用方式

### 消费模型：

### 队列模型：

生产者往某个队列里面发送消息，一个队列可以存储多个消费者的信息。一个队列也可以有多个消费者，但是**消费者之间是竞争关系**，一个消息只能被一个消费者消费。在消息被确认消费过后，会被从消息队列中移除掉，即消费者不能再次消费到已经被消费的数据。



![img](https://img-blog.csdnimg.cn/e688d29017d2440684f276fc8ea3204a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



队列模型

### 发布/订阅模式：

为了解决**一条消息能被多个消费者消费的问题**，发布/订阅模式是个很不错的选择。生产者将消息塞到消息队列对应的topic中，所有订阅了这个topic的消费者都能消费这条消息。

借用看到例子，可以这么理解，发布/订阅模型等于我们都加入了一个群聊中，我发一条消息，加入了这个群聊的人都能收到这条消息。 那么队列模型就是一对一聊天，我发给你的消息，只能在你的聊天窗口弹出，是不可能弹出到别人的聊天窗口中的。

而队列模式实现给群聊中的所有人的发送的方案，采用的是**多队列全量存储**的方式，但是出现数据冗余的情况。简单来说就是一对一聊天中发送同样的消息。

**kafka** 和 **RocketMQ**使用的是发布订阅模式，而**RabbitMQ**使用的是队列模式。



![img](https://img-blog.csdnimg.cn/8c72829166fc4e529867fc6fbbd3469f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



发布订阅模型

### 消息获取方式



![img](https://img-blog.csdnimg.cn/d366997126c142fc82e6e6d227b2ad1f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



kafka消息获取方式

### 生产者

**producer 采用push（推）模式向broker 中写入数据。**

**pull （拉）模式**需要kafka集群事先知晓 producer的信息，即producer需要先注册后才能使用。当有生产者实例宕机了，可能会存在空等。若需要扩展新的producer，则需要先注册，在后续的kafka版本中逐步地和zookeeper进行了解耦，注册成为了一个麻烦的事情。

**push（推）模式**的优点是 生产者有数据就塞给消息队列，不用管其他的事情，只用专注于生产数据。





### 消费者

**consumer 采用 pull（拉） 模式从 broker 中读取数据**。

**push（推）模式很难适应消费速率不同的消费者，因为消息发送速率是由 broker 决定的**。它的目标是尽可能以最快速度传递消息，但是这样很容易造成 consumer 来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而 pull 模式则可以根据 consumer 的消费能力以适当的速率消费消息。

**pull 模式不足之处**是，如果 kafka 没有数据，消费者可能会陷入循环中， 一直返回空数据。 针对这一点， Kafka 的消费者在消费数据时会传入一个时长参数 timeout，如果当前没有数据可供消费， consumer 会等待一段时间之后再返回，这段时长即为 timeout。

**轮询：**

那么消费者是如何知道生产者发送了数据呢？换一句话来说就是，消费者什么时候 pull 数据呢？ 其实生产者产生的数据消费者是不知道的，KafkaConsumer 采用**轮询**的方式定期去 Kafka Broker 中进行数据的检索，如果有数据就用来消费，如果没有就再继续轮询等待。

pull VS push

### 文件存储

先说结论，kafka存储的数据是以追加的方式添加到队列尾部。读写数据是顺序读写。



![img](https://img-blog.csdnimg.cn/a1388f925fc04eee905c4f94a59034e3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



kafka文件存储方式

由于生产者生产的消息会不断追加到 log 文件末尾， 为防止 log 文件过大导致数据定位效率低下， Kafka 采取了**分片**和**索引**机制，将每个 partition 分为多个 segment。

每个 segment对应两个文件——“.index”文件和“.log”文件。 这些文件位于一个文件夹下， 该文件夹的命名规则为： topic 名称+分区序号。例如， first 这个 topic 有三个分区，则其对应的文件夹为 first-0,first-1,first-2。

```
00000000000000000000.index
log
00000000000000170410.index
log
00000000000000239430.index
log
```

index 和 log 文件以当前 segment 的第一条消息的 offset 命名。下图为 index 文件和 log文件的结构示意图。



![img](https://img-blog.csdnimg.cn/dafb7137bd974a3cb2b4b67e04a5eaf9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



segment示意图

**“.index”文件存储大量的索引信息，“.log”文件存储大量的数据**，索引文件中的元数据指向对应数据文件中 message 的物理偏移地址。





### kafka四大API

**Producer API**，它允许应用程序向一个或多个 topics 上发送消息记录

**Consumer API**，允许应用程序订阅一个或多个 topics 并处理为其生成的记录流

**Streams API**，它允许应用程序作为流处理器，从一个或多个主题中消费输入流并为其生成输出流，有效的将输入流转换为输出流。

**Connector API**，它允许构建和运行将 Kafka 主题连接到现有应用程序或数据系统的可用生产者和消费者。例如，关系数据库的连接器可能会捕获对表的所有更改





官方API

### 如何保证数据高可靠、不丢失

### 数据丢失的原因



![img](https://img-blog.csdnimg.cn/85be674dfbd9471197c6811c5b57c8ef.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



消息发送应答

可以看到一共有三个阶段，分别是**生产消息、存储消息和消费消息**。 那么这三个阶段都是有可能丢失消息的。



1. **（生产消息）**如果出现了网络不可用、消息本身不合格等原因导致消息根本没有被 Broker 接收，那就相当于消息在生产者端就消失了。

2. **（存储消息）**Broker 端的消息丢失，一般是由 Broker 服务不可用造成的，例如 Broker 都宕机了导致消息丢失

3. **（消费消息）**消费者在消费消息的过程中，会同时更新消费者位移，也就是「已经消费到哪一条消息了」。这里就存在一个问题，当消费一个消息的时候，是先处理消息，成功后再更新位移，还是先更新位移，再处理消息。

   如果先更新位移，在处理消息，当消息处理出现问题，或者更新完位移、消息还未处理，消费者出现宕机等问题的时候，消息就会丢失。

   而如果先处理消息再更新位移，虽然可能会出现重复消费同一个消息的问题，但是，我们可以通过消费者处理逻辑实现幂等的方式来解决。





### 解决方案

### producer 生产消息

### ack 机制

生产者 acks参数指定了必须要有多少个分区副本收到消息，生产者才认为该消息是写入成功的，这个参数对于消息是否丢失起着重要作用。





### ack 策略

现在我们已经知道生产者发送消息有个确认的机制，那么Kafka里是何时确认呢？Kafka是通过配置acks的值确认机制的，这里一共提供了三种策略，对应不同的ACK机制：

1. acks=0，生产者**不等待broker的响应**。这种情况下**延迟最低**，但是有**可能丢失数据**，比较适合高吞吐量、接受消息丢失的场景。
2. acks=1，生产者发送消息等待broker的响应，等待leader落盘成功后响应确认。这种情况下，如果是在leader完成同步消息给follower前发生故障，则**可能发生消息丢失**。
3. acks=-1，生产者发送消息等待broker的响应，直到leader和follower全部落盘成功后才会响应确认。此机制能严格保证不丢失数据。但当所有的follower同步完成之后，leader发送ack响应之前，leader发生了宕机，此时生产者会以为发送失败了，然后会重新发送数据给新的leader，因此该情况下会导致**数据重复**发送。





### broker 存储消息

存储消息阶段需要在**消息刷盘之后**再给生产者响应，假设消息写入缓存中就返回响应，那么机器突然断电这消息就没了，而生产者以为已经发送成功了。

如果`Broker`是集群部署，有多副本机制，即消息不仅仅要写入当前,还需要写入副本机中。那配置成至少写入两台机子后再给生产者响应。这样基本上就能保证存储的可靠了。**所以broker 消息存储主要是靠的是冗余副本，即多个Replica**





### ISR机制 和 AR机制

简单来说，分区中的所有副本统称为 `AR` (Assigned Replicas)。所有与leader副本保持一定程度同步的副本（包括leader副本在内）组成 `ISR` (In Sync Replicas)。 ISR 集合是 AR 集合的一个子集。消息会先发送到leader副本，然后follower副本才能从leader中拉取消息进行同步。同步期间，follow副本相对于leader副本而言会有一定程度的滞后。前面所说的 ”一定程度同步“ 是指可忍受的滞后范围，这个范围可以通过参数进行配置。于leader副本同步滞后过多的副本（不包括leader副本）将组成 `OSR` （Out-of-Sync Replied）由此可见，**AR = ISR + OSR。正常情况下**，所有的follower副本都应该与leader 副本保持 一定程度的同步，即AR=ISR，OSR集合为空。





**leader副本负责维护和跟踪** 集合中所有follower副本的滞后状态，当follower副本落后太多或失效时，**即follower长时间未向leader发送消息**，leader副本会把它从 ISR 集合中剔除。**如果 集合中所有follower副本“追上”了leader副本，那么leader副本会把它从 OSR 集合转移至 ISR 集合【副本可以在OSP,ISR中来回移动】**。默认情况下，当leader副本发生故障时，只有在 ISR 集合中的follower副本才有资格被选举为新的leader，而在 OSR 集合中的副本则没有任何机会（不过这个可以通过配置来改变）。





### broker恢复机制

- LEO：（Log End Offset）每个副本的最后一个offset
- HW：（High Watermark）高水位，指的是消费者能见到的最大的 offset， ISR 队列中最小的 LEO。可以理解为短板效应

**follower 故障**：follower 发生故障后会被临时踢出 ISR，待该 follower 恢复后， follower 会读取本地磁盘记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 leader 进行同步。等该 follower 的 LEO 大于等于该 Partition 的 HW，即 follower 追上 leader 之后，就可以重新加入 ISR 了。

**leader 故障**：leader 发生故障之后，会从 ISR 中选出一个新的 leader，之后，为保证多个副本之间的数据一致性， 其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader同步数据。





### comsumer 消费消息

消费者拿到消息之后直接存入内存队列中就直接返回给消费成功，这样其实是不算消息消费成功的。我们需要考虑消息放在内存之后消费者就宕机了怎么办，若直接设置为消费成功，当前情况下本条消息相当于丢失了。

所以我们应该在**消费者真正执行完业务逻辑之后，再发送给消费成功**，这才是真正的消费了。





### 如何保证消息有序

有序性分为：**全局有序和局部有序**





### 全局有序

如果要保证消息全局有序，首先只能由一个生产者往Topic发送消息，并且一个Topic内部只能有一个分区（partition）。消费者也必须是单线程的消费数据。这样消息才会是全局有序的。

不过一般情况下，我们不需要全局有序。一般消息的粒度不会很大，例如，同步MySql BinLog 也只需要保证单表消息有序即可。



![img](https://img-blog.csdnimg.cn/e9ca924b6fc4494e8269716f692ccffa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



消息全局有序

### 局部有序

绝大多数的需求的有序性的要求都是局部有序，局部有序我们就可以将Topic内部划分成我们需要的分区数，把消息通过分区策略发往固定的分区中。每个partition对应一个单线程处理的消费者，这样既完成了部分有序的需求，又可以通过partition数量的并发来提高消息处理消息。





### 小结

每个分区内,每条消息都有offset,所以只能在同一分区内有序,但不同的分区无法做到消息顺序性





### 如何保证数据不重复

### 数据重复的原因

- (`Producer` -->) 生产者已经往发送消息了，也收到了消息，并且写入了。当时响应由于网络原因生产者没有收到，然后生产者又重发了一次，此时消息就重复了。
- ( --> `Consumer`)假设我们消费者拿到消息消费了，业务逻辑已经走完了，事务提交了，此时需要更新`Consumer offset`了，然后这个消费者挂了，另一个消费者顶上，此时还没更新，于是又拿到刚才那条消息，业务又被执行了一遍。于是消息又重复了





### 解决方案

可以看到正常业务而言**消息重复是不可避免的**，因此我们只能从**另一个角度**来解决重复消息的问题。我们如何保证消费重复消息后，最终的结果是一样的。

关键点就是**幂等**。既然我们不能防止重复消息的产生，那么我们只能在业务上处理重复消息所带来的影响。





### 幂等性

幂等性定义：

> 用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用。

例如这条 SQL `update t1 set money = 150 where id = 1 and money = 100;` 执行多少遍`money`都是150，这就叫幂等。





### 如何保证消息发送的幂等性



### produce -- > broke

> 每个producer会分配一个唯一 的PID，发往同一个broker的消息会附带一个Sequence Number，broker端会对<PID,partitionId,Sequence Number>做一个缓存，当具有相同主键的消息提交时，Kafka只会持久化一条。





注意：

PID 会随着生产者重启而发生变化，并且不同的partition对应的partitionId也不相同。





### broke ---> comsumer

**具体还需要参照业务细节来实现**。这里提供一个参考，可以通过上面那条 SQL 一样，做了个**前置条件判断**，即`money = 100`情况，并且直接修改，更通用的是做个`version`即版本号控制，对比消息中的版本号和数据库中的版本号。





### 如何处理消息堆积

### 消息堆积的原因

消息的堆积往往是因为**生产者的生产速度与消费者的消费速度不匹配**。有可能是因为消息消费失败反复重试造成的，也有可能就是消费者消费能力弱，渐渐地消息就积压了。





### 解决方案

### 阻塞生产者消息

消费速度跟不上，那么阻塞住生产者不就可以了？ 但是在使用场景中，业务方的数据是源源不断的，阻塞住很有可能带来损失，一般不采用这种方案。





### 增加Topic中partition数量

### 增加消费者数量

1. 消费者数量 < partition的数量， 可以直接增加消费者数量
2. 消费者数量 <= partition的数量，**注意队列数一定要增加**，不然新增加的消费者是没东西消费的。**一个Topic中，一个partition只会分配给一个消费者**。

### 临时队列



![img](https://img-blog.csdnimg.cn/c340f11b2c1d41bbbd40696e0e8839ee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



临时队列

我们可能会遇到这样的一种场景，消费者宕机了好久。等到消费者恢复过来的时候，消息已经堆积成山了。如果还按照以前的速度来进行消费，肯定是不能满足需求的。所以这个时候需要提速消费！！





使用 **临时队列** 是一个不错的选择：

新建一个 Topic，设置为 20 个 Partition

Consumer 不再处理业务逻辑了，只负责搬运，把消息放到临时 Topic 中

这 20 个 Partition 可以有 20个 Consumer 了，它们来处理原来的业务逻辑。





### 如何保证数据的一致性

数据的高可用性通常采用的是数据冗余的方式来实现的，而强一致性和高可用性相对应。一致性需要保证副本之间的同步。

### LEO 和 HW



![img](https://img-blog.csdnimg.cn/4bd1b81eaaff42be8194b74baca5ee10.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



LEO和HW

### 分区分配策略

### 分区的原因

1. **方便在集群中扩展**，每个 Partition 可以通过调整以适应它所在的机器，而一个 topic又可以有多个 Partition 组成，因此整个集群就可以适应适合的数据了；
2. **可以提高并发**，因为可以以 Partition 为单位读写了。





### 生产者分区机制

Kafka 对于数据的读写是以`分区`为粒度的，分区可以分布在多个主机（Broker）中，这样每个节点能够实现独立的数据写入和读取，并且能够通过增加新的节点来增加 Kafka 集群的吞吐量，通过分区部署在多个 Broker 来实现`负载均衡`的效果。

由于消息是存在主题（topic）的分区（partition）中的，所以当 Producer 生产者发送产生一条消息发给 topic 的时候，你如何判断这条消息会存在哪个分区中呢？ 分区策略就是用来解决这个问题的。





### 分区策略

Kafka 的分区策略指的就是将生产者发送到哪个分区的算法。





### 指定partition

指明partition时，直接将该值作为partition值。

### 随机轮询

![img](https://img-blog.csdnimg.cn/43afc0c2643642f39cd43b64876c246f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



随机轮询

### 按key存储

若未指明，但是有key的情况下，将key的hash值与该topic下可用的分区数取余得到partition值。



![img](https://img-blog.csdnimg.cn/fb8dc0e1ae524036a7b1f329210607da.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



按key hash存储

### 顺序轮询（round-robin）

若既未指明partition，也没有key时，在第一次调用时随机生成一个整数（后续每次调用都会在这个整数上自增），将该整数与topic下可用的分区数取余得到partition值，也就是常说的`round-robin`算法。



![img](https://img-blog.csdnimg.cn/393e8f92d1e84821b7fb368b7c0e395c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



顺序轮询

### 消费者如何和Parition相对应

### Rebalance 消费者再平衡机制

所谓的再平衡，指的是在kafka consumer所订阅的topic发生变化时发生的一种分区重分配机制。一般有三种情况会触发再平衡：

- consumer group中的**新增或删除**某个consumer，导致其所消费的分区需要分配到组内其他的consumer上；
- consumer订阅的topic发生变化，比如订阅的topic采用的是正则表达式的形式，如`test-*`此时如果有一个新建了一个topic `test-user`，那么这个topic的所有分区也是会自动分配给当前的consumer的，此时就会发生再平衡；
- consumer所订阅的topic发生了新增分区的行为，那么新增的分区就会分配给当前的consumer，此时就会触发再平衡。

Kafka提供的再平衡策略主要有三种：`Round Robin`，`Range`和`Sticky`，默认使用的是。这三种分配策略的主要区别在于：

- ：会采用轮询的方式将当前所有的分区依次分配给所有的consumer；
- ：首先会计算每个consumer可以消费的分区个数，然后按照顺序将指定个数范围的分区分配给各个consumer；
- ：这种分区策略是最新版本中新增的一种策略，其主要实现了两个目的：
  - 将现有的分区**尽可能均衡**的分配给各个consumer，存在此目的的原因在于和分配策略实际上都会导致某几个consumer承载过多的分区，从而导致消费压力不均衡；
  - 如果发生再平衡，那么重新分配之后在前一点的基础上会尽力保证当前未宕机的consumer所消费的分区不会被分配给其他的consumer上；





### Round Robin

关于Roudn Robin重分配策略，其主要采用的是一种轮询的方式**分配所有的分区**，该策略主要实现的步骤如下。这里我们首先假设有三个topic：t0、t1和t2，这三个topic拥有的分区数分别为1、2和3，那么总共有六个分区，这六个分区分别为：t0-0、t1-0、t1-1、t2-0、t2-1和t2-2。这里假设我们有三个consumer：C0、C1和C2，它们订阅情况为：C0订阅t0，C1订阅t0和t1，C2订阅t0、t1和t2。那么这些分区的分配步骤如下：

- 首先将所有的partition和consumer按照字典序进行排序，所谓的字典序，就是按照其名称的字符串顺序，那么上面的六个分区和三个consumer排序之后分别为：

![img](https://img-blog.csdnimg.cn/b963fbac7c174f649c93edb3f3962317.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



- 然后依次以按顺序轮询的方式将这六个分区分配给三个consumer，如果当前consumer没有订阅当前分区所在的topic，则轮询的判断下一个consumer：
- 尝试将t0-0分配给C0，由于C0订阅了t0，因而可以分配成功；
- 尝试将t1-0分配给C1，由于C1订阅了t1，因而可以分配成功；
- 尝试将t1-1分配给C2，由于C2订阅了t1，因而可以分配成功；
- 尝试将t2-0分配给C0，由于C0没有订阅t2，因而会轮询下一个consumer；
- 尝试将t2-0分配给C1，由于C1没有订阅t2，因而会轮询下一个consumer；
- 尝试将t2-0分配给C2，由于C2订阅了t2，因而可以分配成功；
- 同理由于t2-1和t2-2所在的topic都没有被C0和C1所订阅，因而都不会分配成功，最终都会分配给C2。
- 按照上述的步骤将所有的分区都分配完毕之后，最终分区的订阅情况如下：

![img](https://img-blog.csdnimg.cn/e8b79d60d46c4fa68d5b881b6cb54bb1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



从上面的步骤分析可以看出，轮询的策略就是简单的将所有的partition和consumer按照字典序进行排序之后，然后依次将partition分配给各个consumer，如果当前的consumer没有订阅当前的partition，那么就会轮询下一个consumer，直至最终将所有的分区都分配完毕。但是从上面的分配结果可以看出，

轮询的方式会导致每个consumer所承载的分区数量不一致，从而导致各个consumer压力不均一。





### Range

所谓的Range重分配策略，就是首先会计算各个consumer将会承载的分区数量，然后将指定数量的分区分配给该consumer。这里我们假设有两个consumer：C0和C1，两个topic：t0和t1，这两个topic分别都有三个分区，那么总共的分区有六个：t0-0、t0-1、t0-2、t1-0、t1-1和t1-2。那么Range分配策略将会按照如下步骤进行分区的分配：

- 需要注意的是，Range策略是**按照topic依次进行分配**的，比如我们以t0进行讲解，其首先会获取t0的所有分区：t0-0、t0-1和t0-2，以及所有订阅了该topic的consumer：C0和C1，并且会将这些分区和consumer按照字典序进行排序；

- 然后按照平均分配的方式计算每个consumer会得到多少个分区，如果没有除尽，则会将多出来的分区依次计算到前面几个consumer。比如这里是三个分区和两个consumer，那么每个consumer至少会得到1个分区，而3除以2后还余1，那么就会将多余的部分依次算到前面几个consumer，也就是这里的1会分配给第一个consumer，总结来说，那么C0将会从第0个分区开始，分配2个分区，而C1将会从第2个分区开始，分配1个分区；

- 同理，按照上面的步骤依次进行后面的topic的分配。

- 最终上面六个分区的分配情况如下：

  



![img](https://img-blog.csdnimg.cn/587201b5bd454d61ba669d266d477571.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_16,color_FFFFFF,t_70,g_se,x_16)



可以看到，如果按照

分区方式进行分配，其本质上是依次遍历每个topic，然后将这些topic的分区按照其所订阅的consumer数量进行平均的范围分配。这种方式从计算原理上就会导致排序在前面的consumer分配到更多的分区，从而导致各个consumer的压力不均衡。





### Sticky

策略是新版本中新增的策略，顾名思义，这种策略会保证**再分配时已经分配过的分区尽量保证其能够继续由当前正在消费的consumer继续消费**，当然，前提是每个consumer所分配的分区数量都大致相同，这样能够保证每个consumer消费压力比较均衡。





### 消费者初始分配

初始分配使用的就是`sticky`策略，初始状态分配的特点是，所有的分区都还未分配到任意一个consumer上。

这里我们假设有三个consumer：C0、C1和C2，三个topic：t0、t1和t2，这三个topic分别有1、2和3个分区，那么总共的分区为：t0-0、t1-0、t1-1、t2-0、t2-1和t2-2。关于订阅情况，这里C0订阅了t0，C1订阅了t0和1，C2则订阅了t0、t1和t2。这里的分区分配规则如下：

- 首先将所有的分区进行排序，排序方式为：首先按照当前分区所分配的consumer数量从低到高进行排序，如果consumer数量相同，则按照分区的字典序进行排序。这里六个分区由于所在的topic的订阅情况各不相同，因而其排序结果如下：

  

![img](https://img-blog.csdnimg.cn/12956c5d8e9142ffb43ade4d848a1634.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



分区排序结果

- 然后将所有的consumer进行排序，其排序方式为：首先按照当前consumer已经分配的分区数量有小到大排序，如果两个consumer分配的分区数量相同，则会按照其名称的字典序进行排序。由于初始时，这三个consumer都没。
- 然后将各个分区依次遍历分配给各个consumer，首先需要注意的是，这里的遍历并不是C0分配完了再分配给C1，而是每次分配分区的时候都整个的对所有的consumer从头开始遍历分配，如果当前consumer没有订阅当前分区，则会遍历下一个consumer。然后需要注意的是，在整个分配的过程中，**各个consumer所分配的分区数是动态变化的**，而这种变化是会体现在各个consumer的排序上的，比如初始时C0是排在第一个的，此时如果分配了一个分区给C0，那么C0就会排到最后，因为其拥有的分区数是最多的，**即始终按照所含分区数量从小到大排序**。上面的六个分区整体的分配流程如下：
- 首先将t2-0尝试分配给C0，由于C0没有订阅t2，因而分配不成功，继续轮询下一个consumer；
- 然后将t2-0尝试分配给C1，由于C1没有订阅t2，因而分配不成功，继续轮询下一个consumer；
- 接着将t2-0尝试分配给C2，由于C2订阅了t2，因而分配成功，此时由于C2分配的分区数发生变化，各个consumer变更后的排序结果为：

![img](https://img-blog.csdnimg.cn/4182b0c17b9941b4b4b70d65c9c065b7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



接下来的t2-1和t2-2，由于也只有C2订阅了t2，因而其最终还是会分配给C2，最终在t2-0、t2-1和t2-2分配完之后，各个consumer的排序以及其分区分配情况如下：

![img](https://img-blog.csdnimg.cn/a7ed40172f3d44088603b549f6f5d6bd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



- 接着继续分配t1-0，首先尝试将其分配给C0，由于C0没有订阅t1，因而分配不成功，继续轮询下一个consumer；

- 然后尝试将t1-0分配给C1，由于C1订阅了t1，因而分配成功，此时各个。

- 同理，接下来会分配t1-1，虽然C1和C2都订阅了t1，但是由于C1排在C2前面，因而该分区会分配给C1，即：

  

![img](https://img-blog.csdnimg.cn/5b82340102614a5b943a9435d8d966dc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



上面的分配过程中，需要始终注意的是，虽然示例中的consumer顺序始终没有变化，但这是由于各个分区分配之后正好每个consumer所分配的分区数量的排序结果与初始状态一致。这里读者也可以比较一下这种分配方式与前面讲解的进行对比，可以很明显的发现，重分配策略分配得更加均匀一些。





### kafka日志压缩

压缩一词简单来讲就是一种互换思想，它是一种经典的用 CPU 时间去换磁盘空间或者 I/O 传输量的思想，希望以较小的 CPU 开销带来更少的磁盘占用或更少的网络 I/O 传输。

Kafka 的消息分为两层：消息集合 和 消息。一个消息集合中包含若干条日志项，而日志项才是真正封装消息的地方。Kafka 底层的消息日志由一系列消息集合日志项组成。Kafka 通常不会直接操作具体的一条条消息，它总是在消息集合这个层面上进行`写入`操作。

在 Kafka 中，压缩会发生在两个地方：Kafka Producer 和 Kafka Consumer，为什么启用压缩？说白了就是消息太大，需要`变小一点` 来使消息发的更快一些。

**有压缩必有解压缩**，Producer 使用压缩算法压缩消息后并发送给服务器后，由 Consumer 消费者进行解压缩，因为采用的何种压缩算法是随着 key、value 一起发送过去的，所以消费者知道采用何种压缩算法。



官方解释

### 为什么kafka速度快？

Kafka 实现了`零拷贝`原理来快速移动数据，避免了内核之间的切换。Kafka 可以将数据记录分批发送，从生产者到文件系统（Kafka 主题日志）到消费者，可以端到端的查看这些批次的数据。

批处理能够进行更有效的数据压缩并减少 I/O 延迟，Kafka 采取顺序写入磁盘的方式，避免了随机磁盘寻址的浪费。



![img](https://img-blog.csdnimg.cn/d25d817c72f8476ea234111d1156d248.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6LSd5ouJ576O,size_20,color_FFFFFF,t_70,g_se,x_16)



零拷贝

总结一下其实就是四个要点

- 顺序读写
- 零拷贝
- 消息压缩
- 分批发送