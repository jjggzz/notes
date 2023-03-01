# indexFile

消息消费有commitlog文件和consumerqueue文件就可以完成了。rocketMQ还提供了通过key进行消息查询的功能，这个功能是通过index目录下的indexFile进行索引实现的。这个文件的写入是消息被发送到broker时写入的，当然，**如果消息没有指定key，则不写入**

## indexFile结构

文件命名以创建时的时间戳命名。每个文件由三部分组成，indexHeader，slot槽、indexes索引数据，每个indexFile有500万个slot槽。

indexHeader包含如下数据：

beginTimestamp：文件中第一条消息的存储时间

endTimestamp：文件中最后一条消息存储时间

beginPhyoffset：文件中第一条消息在commitlog中的commitlog offset

endPhyoffset：文件中最后一条消息在commitlog中的commitlog offset

hashSlotCount：已填充有index的slot数量

indexCount：文件中的索引个数

在物理上indexes索引数据存放在slot槽的后面，slot槽每个4字节，存储的是indexes索引数据的偏移量。indexes中除了indexData外还有一个preIndexNo域，它存储前一个与他拥有相同slot槽的索引条目的位置。

简单可以**理解为一个hashTable**。

1. 每当有指定key的消息需要写入，首先计算key的hash值，然后对500w取余，得到slot槽位
2. 如果该slot上已经存在着值，那么将自己的preIndexNo指向它，并将自己的索引位置写入到该slot
3. 这样就形成了一个500w数组+链表的结构

indexes条目固定20字节：

4字节keyHash，

8字节commitlog offset，

4字节当前key存储时间与文件创建时间的差值，

4字节存储preIndexNo域

**要根据key进行数据查询，首先计算key的hash值与500w取余，定位到slot，然后就可以知道最新的索引条目在哪，然后进行读取，不符合则读取preIndexNo域，直到找到为止**

## 以时间戳命名文件有啥用？

首先需要明白indexFile何时创建：

1. 当第一条带key的消息发送到broker，系统发现没有indexFile就会创建一个
2. 当文件中索引条目数达到2000w时，再尝试写入则创建新的

在查询时如果有时间条件，可以快速排除一部分文件，减少时间消耗。**按照时间要求快速定位到文件之后再计算key的hash值与500w取余，定位到slot，然后就可以知道最新的索引条目在哪，然后进行读取，按照当前key存储时间与文件创建时间的差值来判断是否是符合条件的数据。**