# rebalance

rebalance是指一个topic的多个queue在被某个消费者组消费的过程中**消费者再平衡的过程**，比如说4个queue被某个消费者组中的4个消费者消费，每个消费者会分到一个，突然挂掉了一个消费者，此时会产生rebalance。再或者本来4个queue只有一个消费者消费，然后又新增了一个消费者。

## rebalance限制

由于一个queue对于某个消费者组而言只能被其组下的一个消费者消费，所以如果消费者数量大于queue数量则**多出来的消费者无作用**

## rebalance的危害

在rebalance时，虽然可以提升其消费能力，但是也会带来一些问题：

消费暂停：在只有一个consumer时，它负责消费所有的queue，新增consumer从而触发rebalance，此时原consumer会暂停部分queue的消费，等到重新分配完成后这些queue才能被继续消费

消费重复：消息消费与offset的提交**默认是异步的**，consumer在消费分配给自己的queue时必须接着之前consumer消费进度的offset继续消费，由于异步提交到broker的关系，有可能造成重复消费（consumer1已经消费到10，而提交给broker的还是8，有两条还未提交，此时发生rebalance，consumer2会从8开始消费，所以出现重复消费）

消费突刺：由于rebalance会导致重复消费，如果重复消费的消息过多，或者rebalance期间积压了很多消息，则会出现尖刺现象

## rebalance的原因

rebalance的原因一般来说有两个：消费者数量发生变化、queue发生扩容或者缩容、broker升级运维

## rebalance过程

broker中维护着多个map集合，这些集合中存储着当前topic的queue信息、消费者组中消费者实例的信息。一旦这些信息发生变化，立刻向消费者组中的所有消费者实例发送rebalance通知。

> topicConfigManager：key是topic名称，value是TopicConfig，TopicConfig中维护着该topic下的所有queue信息

> comsumerGroupManager：key是Comsumer Group Id，value是ConsumerGroupInfo，ConsumerGroupInfo中维护着该消费者组中的所有消费者实例

> consumerOffsetManage：key是topic和订阅该topic消费者组的组合，value是一个map，内层map的key是queueId，value为该queue的消费进度offset

> 还有一个map维护了consumer与topic的queue的订阅关系

消费者在接受到通知后采用**queue分配算法**获取到自己相应的queue，所以rebalance是由客户端自己完成的（kafka虽然也是在客户端完成，但是会由broker选出一台机客户端做为leader然后让它进行分配最后上报，然后通过broker同步给其他客户端，rocketMQ是每台机器自己算）

## queue分配算法

#### 平均分配

根据queueCount / consumerCount计算出平均值，如果能够整除则按照顺序进行逐个分配，如果不能则将多余的queue按照comsuner顺序进行逐个分配（按照该算法计算好应该分配多少个queue之后再去进行队列的获取）

如果有10个queue，3个consumer，则：

comsuner1获取到的是1，2，3，4

comsuner2获取到的是5，6，7

comsuner3获取到的是8，9，10

#### 环型平均

根据消费者顺序依次在由queue环上进行分配，如果有10个queue，3个consumer，则：

comsuner1获取到的是1，4，7，10

comsuner2获取到的是2，5，8

comsuner3获取到的是3，6，9

两种算法简单，分配平均，但是在是如果消费者数量发生变化引起的rebalance则消费关系会发生较大的变化

#### 一致性hash

将comsumer的hash值放到hash环上作为node，然后依次计算queue对应的hash值落在环上的位置。然后顺时针遇到第一个node就作为其消费者。

这种有可能造成偏斜，但是如果消费者数量发生变化引起的rebalance则则消费关系不会发生较大的变化

#### 同机房策略

优先分配给同机房的消费者，如果不存在则按照平均分配或者环型平均策略进行分配

## 至少一次

rocketMQ要求每条消息至少消费一次。comsumer在消费完一条消息之后会向其**消费进度记录器**提交消费消息的offset，只有被记录成功才能被称为成功消费

> 什么是消费进度记录器？
>
> 广播模式，comsumer本身就是消费进度记录器
>
> 集群模式，broker是消费进度记录器