#### 微服务架构中的消息队列

微服务时代来临，互联网应用逐渐由单体应用架构转变为分布式的微服务架构。各个微服务之间的消息通信有同步和异步之分，而微服务化使得同步调用不再具有低延时和高可用的特点。为了保证用户体验，我们不得不将部分同步操作转变为异步，使消息队列的设计在异步的消息架构中变得至关重要。

一起来探讨一下，我们如何保证消息准确送达？业界这么多消息队列框架，如何挑选最合适自己业务模型的？

#### Outline

---

##### 为什么大街小巷都在谈微服务？Why microservice?

因为：

1. 社会在不断向前发展
2. 业务需求迅速增加
3. 进行容量扩展
   * 垂直扩展（Scale Up）成本高昂，可预见的上限
   * 水平拓展（Scale Out）成本较为低廉
4. 水平拓展时
   * 单体应用（Monolithic）紧耦合，扩展的时候效率不按比例，粗犷
   * 微服务（Microservice）松耦合，细粒度，高效率


---

##### 解耦 Decoupling

> 劳动生产力最大的进步，以及劳动在任何地方的运用中体现的大部分的技能、熟练度和判断力似乎都是分工的结果。-《国富论》

解耦的过程：

* 是治疗患有重度洁癖强迫症的架构师的过程；
* 是将披萨分成一片一片的过程，却又享受地看着香浓芝士在其间相连的过程；
* 是将业务一步一步剥离开来，或把不那么核心的业务过程抛到外围的过程。

比如：


> 在老马家的 Web 端购买了一单商品后， 网页显示支付成功，App 在半分钟以后收到同样的消息推送。

> 负责订单处理开发的同学对开发付款接口的说：“随你怎么处理，保证用户把款付了就行。“

为了解耦，我们可以将一个业务流程分拆开来，每个部分由不同的线程或是服务来单独完成。

而要完成这个业务事件的传递，我们必须引入消息通信这个概念。

在这两种架构中，我们在某些业务场景下会有线程间（单体应用）或者服务间（微服务）的通信，这些通信根据业务的特性被分成同步和异步两种。

---

##### 同步和异步通信 Synchronous vs. Asynchronous

同步形式的通信，即各种形态的 Remote Procedure Call：

* Google's gRPC
* Apache Thrift
* Alibaba's Dubbo
* Plain RESTFul API


异步形式的通信：

* **消息队列**
* 其它形式的消息的队列
  * 数据库
  * 文件


以及各种形式的序列化方法：

- Protocol Buffer
- JSON
- MessagePack



分布式RPC框架性能大比拼

http://colobu.com/2016/09/05/benchmarks-of-popular-rpc-frameworks/

------

##### 消息队列架构 Message Queue Topology

三类角色：

Sender -> Message Broker -> Receiver

普通的点对点模型。

Producer -> Message Broker -> Consumer

当 Sender 和 Receiver 为多个节点时，任一 Sender 发送的消息可以被任一 Receiver 接收时，就变成了生产者消费者模型。

Publisher -> Message Broker -> Subscriber

当 Message Broker 会将 Sender 发送的消息分类，路由到不同的队列并由相应的 Receiver 接收时，就变成了发布订阅模型。

广义地来说以下都是消息队列的形式：

- HTML5 Web Worker
  - <u>当主线程调用 postMessage(msg) 时，WebKit 会把 msg 存入内存中的临时队列，当 worker 线程准备完毕时，再从队列中取得消息开始处理。</u>
- 数据库中的一张表
  - <u>A 服务把注册完成的用户手机号码填入数据库中的一张表，B 服务每隔几分钟从这张表中取出一部分手机号码进行短信推送。</u>

---

##### 现实世界的抉择 Make your choice

> Distributed systems are all about trade-offs.

* 发送者需要知道准确运行结果 -> 强一致性 -> 同步
  * 通过事务保证数据准确性
  * 分布式架构中时效性低下
* 发送者不在乎运行结果 -> 弱一致性/最终一致性 -> 异步
  * 最终一致性 - "Everything will be okay."
    * <u>一致性就是指系统状态的一致，比如一个订单在 A 处处于已支付状态，而在 B 处是未支付状态，这就是不一致状态。而最终一致性保证的是，在非理想情况下，系统中总会出现延迟或者异常，而系统最终的状态是一致的，即这个订单在两个服务内看起来状态是同样的。</u>
  * 一致性程度由框架和业务代码决定
    * <u>部分框架是保证消息的高可靠性，在业务层面几乎不可能丢失数据</u>

---

##### 消息确认 Acknowledgements and Confirms

分布式应用比起传统单体应用有一个较大的劣势，就是传输信息的媒介不再可靠，网络链接传输断开也更加容易发生。消息队列与任一方通信的过程中，网络链接出错的瞬间，谁也不知道数据包传输到哪里了，要么正在被序列化或反序列化，要么还在操作系统的 Buffer 里，甚至还可能在交换机里。总而言之，这个数据包的使命已经 Gameover 了，而上层的机制需要保证同样的数据包还要再传输一次。

当然，我们会想到，这不是 TCP 做的事情么，保证传输链路的可靠性，但这只是网络层的事情。而消息队列是上层建筑，远水救不了近火，消息队列只能自己动手来保证了，所以就引入了消息确认（Acknowledgements）。

消息确认像是 TCP 中的 ACK，但它更加负责。只有当 Receiver 确认收到了消息，并且开始对消息负责了才可以发送消息确认。这里所说的负责一般是指确认持久化完成了、已经转发给下一级了或其他处理完成了。所以当 Sender 接受到了消息确认以后就可以放心地删除消息了。

 有了消息确认机制以后，消息队列可以保证 at-least-once delivery 了，当然，也意味会有重复，如果我们关闭消息确认机制，那么消息队列只能保证 at-most-once delivery，简单来说就是零或一次，不靠谱。

幂等操作不是纯函数，纯函数不改变外部状态，幂等是状态机一样的模型。

* 消息确认 Acknowledgements
  * Receiver 告诉 Broker 接收完/处理完消息
  * Broker 告诉 Sender 接收完/处理完消息，这也通常称为 Confirm
  * 保证了 at-least-once delivery
    * 狂发消息给接收者，直到收到消息确认
    * 消息可能重复接收，业务操作需要保证是幂等操作 Idempotent Operation
    * 如果做不到幂等操作，需要做去重过滤
  * 如果没有消息确认，只能保证 at-most-once delivery

Reliability Guide

https://www.rabbitmq.com/reliability.html

Message Delivery Reliability

http://doc.akka.io/docs/akka/snapshot/general/message-delivery-reliability.html

You Cannot Have Exactly-Once Delivery

http://bravenewgeek.com/you-cannot-have-exactly-once-delivery/

---

##### 推还是拉 Push and Pull

推模式，Broker 一收到消息就会给 Receiver 推送：

* 如果 Receiver 处理消息速度慢，而消息持续产生，发送给 Receiver 时，Receiver 并不能处理消息，会返回 reject 或者 error。除非在 Broker 端有记录 Receiver 的状态信息，但无疑会增加系统复杂性。

拉模式，Receiver 主动向 Broker 请求消息：

* 类似于 Web 中的轮询，很多请求是白费功夫
* 可以按需地拉取数据
* 定时拉取，无论是动态调整时间间隔还是指定静态时间间隔，无非就是网络开销和系统延时之间的权衡

长轮询 Long-Polling，Receiver 主动请求消息，Broker 只在有消息时或超时后返回：

* 服务器保持连接会消耗一定资源
* 接近于实时
* 需要设置一个超时重试
* Kafka/RocketMQ

---

##### 顺序消息 Message Order

* 为了保证业务的一致性和实现事务，消息系统难点之一在于消息有序性
* 只有前一个消息得到消息确认后才发送后一个消息
* 不同的消息区分不同的分区 Partition 以提高消息系统效率
* 单实例部署保证每次只有一个线程操作一个 Partition
* 单实例部署导致低可用性问题

---

##### RabbitMQ: Messaging that just works

* Advanced Message Queuing Protocol (AMQP) 实现
  * 一个开放的高级消息队列协议
  * 为通用消息队列架构提供通用的构建工具
  * 满足协议的不同客户端可以互相通信，此外还有 Apache ActiveMQ
* 支持多种模式
  * Point-to-point：点对点，只是用来解耦
  * Work queues：分配任务，通常一个 Sender 发布消息，多个 Receiver 处理消息
  * Pub/Sub：引入消息中转 Exchange，Exchange 中采用 fanout 规则将消息转发给每一个 Receiver
  * Routing：采用 direct 规则，发布消息是可指定 Routing Key，供 Exchange 转发到对应的队列中
  * Topics：采用 topic 规则，发布消息时指定 Topic，以通配符的方式将消息转发到不同队列中
  * RPC：同步模式，阻塞直到获得结果

消息持久化，消息确认可以配置，支持两种模式高可用，A/A 和 A/P

---

##### Apache Kafka: Distributed Message Queue

* LinkedIn 开源的分布式消息队列
* 每个 Topic 的内容被根据一定算法被分配到不同分区 Partition，每个 Partition 内部消息保证有序，并发数量与分区数成正比
* 主要解决单点故障问题，采用分布式架构，由 ZooKeeper 提供高可用性
  - 采用 Active/Passive 模式，每个分区 Partition 有多个备份 Replicas，其中一份为 Leader 承载消息读写，由 ZooKeeper 选举产生，其余的是 Follower，只同步消息。
  - 虽然是单实例，但是这个单实例故障后会有新的 Leader 接管
* 将一定时间内的消息持久化到硬盘中，支持重放 Replay
* 采用顺序读写 Linear Reads and Writers 加速消息读写，用追加 log 文件实现，每条消息在文件中的位置为 Offset，唯一标识每条消息

消息确认可配置。

Kafka 0.10.0 Documentation

http://kafka.apache.org/documentation.html#design

Kafka Partitions Problem

https://github.com/alibaba/RocketMQ/blob/master/wiki/kafka_partitions_problem.md

---

##### ZeroMQ: Brokerless Message Queue

* 歌颂了 Broker 的优点
  * Sender 和 Receiver 无关性：A 不需要知道 B 所在，可以通过 Broker 来寻找 B 所在。
  * Sender 和 Receiver 生命周期不需要重叠：A 发出消息后，就可以关机。
  * 一定程度上减少业务故障：A 发出消息后，宕机，但消息还在，B 业务正常处理。
* 吐槽了 Broker 的缺点
  * 父业务需要调用子业务，甚至孙子业务时，考虑到请求和返回是双向的，内网网络链接数量非常大。
  * 即使采用流水线形式（Pipelined Fashion）的业务调用，由于 Broker 的存在，也好不到哪里去。
  * 由于 Broker 是中心化节点一样的存在，在业务量较大时，Broker 成为整个架构的瓶颈。
* 提出了神思路：去掉 Broker
  * 用目录服务（Directory Service）代替 Broker，类似服务注册与发现模块
  * Sender 用目录服务找到 Receiver，进行直接通信
* 可选地加入了 Broker 的优点：
  * 在中心化的目录服务基础上，加上分布式的 Broker，分布于每对 Sender 组和 Receiver 组之间，为了实现第二和第三个优点
  * 顺便又把中心化的目录服务做成分布式的，避免了单点故障
* 总结：
  * Brokerless 模式下性能超高
  * 高度可定制化，可以搭配多种模式使用
  * 信息默认存在内存中

ZeroMQ Frequently Asked Questions

http://zeromq.org/area:faq

Message Queue Shootout!

http://mikehadlow.blogspot.com/2011/04/message-queue-shootout.html

Broker vs. Brokerless

http://zeromq.org/whitepapers:brokerless

---

##### 其他消息队列 Other Message Queues

* RocketMQ:  Alibaba's MQ, also aliyun ONS.
  * 部分性能比 Kafka 更优，见文档
* NSQ: A realtime distributed messaging platform
  * messages are not durable (by default)
  * messages are delivered at least once
  * messages received are un-ordered
  * consumers eventually find all topic producers

RocketMQ vs. Kafka

https://github.com/alibaba/RocketMQ/blob/master/wiki/rmq_vs_kafka.md

分布式开放消息系统(RocketMQ)的原理与实践

http://www.jianshu.com/p/453c6e7ff81c

NSQ FEATURES & GUARANTEES

http://nsq.io/overview/features_and_guarantees.html

---

##### 总结 Summary

选型时需要考虑：

* 操作是否关心结果
* 是否需要消息确认来保证 at-least-once delivery
* 牺牲网络损耗还是系统延时，选择 Push/Pull/Long-Polling
* 需要的并行程度，决定分区数
* 高可用模式的选择

Dissecting Message Queues

http://bravenewgeek.com/dissecting-message-queues/

Top 10 Uses For A Message Queue

https://www.iron.io/top-10-uses-for-message-queue/