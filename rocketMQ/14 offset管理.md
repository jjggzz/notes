# offset管理

​	这里说的offset是指消费者消费某个队列时的消费进度。由于分为两种模式：**集群模式**和**广播模式**所以分开讨论

## 广播模式本地管理

当消费模式为广播模式的时候，offset使用本地模式存储。相关offset会存储到用户主目录下的.rocketmq_offsets/${clientId}/\${group}/Offsets.json

## 集群模式远程管理

当消费模式为集群模式的时候，offset使用远程模式存储。存储在broker用户主目录的的store/config/consumerOffset.json。集群模式使用远程模式存储主要是为了rebalance后新consumer能够继续消费。

在comsumer启动后消费的第一条信息有三种选择，分别通过枚举值来设置：

1. CONSUMER_FROM_LAST_OFFSET，从队列的最后一条开始
2. CONSUMER_FROM_FIRST_OFFSET，从队列的第一条开始
3. CONSUMER_FROM_TIMESTAMP，从指定的时间开始消费

当消息消费完成，consumer提交offset个broker，broker将其保存到自己的双层map中并写入consumerOffset.json文件中，然后向consumer发送ACK。ACK包含三项数据：当前消费队列的最小offset、当前消费队列的最大offset、下次消费的起始offset

## offset的同步提交与异步提交

集群模式下offset的提交方式有两种：

同步提交：消费者消费完一批消息后向broker提交这些消息的offset，只有等broker返回了ack之后才继续消费下一批，提交失败会重试，等待时是阻塞的

异步提交：消费者消费完一批消息后向broker提交这些消息的offset，但不等待broker响应，可以继续获取下一批消息，这种方式吞吐量得到提高，broker收到offset请求处理完成之后还是会向客户端响应ack