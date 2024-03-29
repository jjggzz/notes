# redis多线程演化

## redis的单线程是指的什么？

redis的单线程指的是网络IO和键值对的读写是由一个线程来处理的。redis在处理客户端的请求时，包括获取（socket读）、解析、执行、内容返回（socket写）等都是由一个顺序串行的主线程处理。这就是所谓的单线程。

redis采用reactor模式的网络模型，对于一个客户端请求主线程负责一个完整的处理过程。

redis的持久化、集群通信等等都是由异步线程完成的。

**综上所述，redis处理请求是单线程的，但是整个系统内部还是多线程的**

## Redis为什么选择单线程？

这个问题并不严谨，在redis的不同版本架构是不一样的，就像java的泛型一样也是在1.5之后支持的。

**在redis3.x这个版本中redis是单线程的**。单线程有一些多线程没有的优势。并且在单线程的情况下依旧很快，**redis的性能瓶颈不在cpu，在机器的内存和网络带宽**（作者原话）

1. IO多路复用+非阻塞IO，业务逻辑使用单线程完成，不需要线程上下文切换
2. 单线程，**不存在资源竞争**，避免了锁的开销
3. redis数**据存在内存，所以单核也够用**，影响redis性能的是内存容量和网络带宽
4. 精心设计的**数据结构，访问效率大部分在O(1)**非常快
5. 在现如今**高并发环境，即使单机使用上多线程技术，也不见得有效**。采用集群化，每个节点自己是单线程，为外部提供服务也是不错的

在**redis4.x以后的版本就不是严格意义上的单线程了**。它引入了一些后台线程做数据删除这些操作，但是处理请求依旧是单线程的。

在**redis6.x以后的版本就不是单线程的了，用一种全新的多线程来解决问题**。

## 为什么redis单线程好好的后面又要弄多线程呢？

一个终极原因是因为单线程在处理数据时天生是阻塞的，即上一个请求未处理完，则无法处理下一个请求，假如某个请求**删除一个大key，非常耗时，则其他请求被阻塞**，只能等待。这种情况会将服务器拖死，完全不能接受。

对于大key删除会造成主线程卡顿的问题，redis之父采用了**lazy free**的方式，将cost比较高的操作从主线程剥离，由bio子线程完成，极大的减少了主线程的阻塞时间。就像unlink命令（4.x以上可用），本质上是后台完成真正的删除操作

## redis多线程演化

由于redis的性能瓶颈主要在内存和网络IO，而内存的提升较为容易，而网络IO则需要一个好的模型来提升其性能。

redis的网络IO模型主要是多路复用，即reactor设计模式。IO多路复用，简单来说就是通过检测文件的读写事件再通知线程执行相关操作，保证redis的非阻塞IO能够顺利执行完成的机制。**多路指的是多个socket链接，复用指的是复用一个线程**，linux下多路复用主要由有三种技术：select、poll、epoll

采用IO多路复用可以让单个线程高效的处理多个连接请求，再加上redis在内存中操作数据非常快，强强联合造成了redis的超高吞吐量。

redis的IO读写是阻塞的，在redis3.x中请求的处理都是由主线程完成，此时线程会同步的将数据从内核空间拷贝到用户空间，并交给redis使用，数据越多拷贝需要的时间就越多。

redis4.x对于大key删除等耗时操作，为了减少主线程被阻塞的时间，采用后台删除的策略，开始慢慢引入多线程了。

redis6.x以后为了进一步提高IO的读写性能，它主要实现思路是将主线程的IO读写任务拆分给一组独立的线程去执行，这样多个socket就可以并行了。采用IO多路复用技术让单个线程高效的处理连接请求，将耗时的**socket读取、解析、写入用另外的线程来完成**。剩下的**命令执行仍然由主线程串行的和内存交互**，保证不会有并发问题

**Redis6.x默认情况下多线程是关闭的。它由以下两个参数控制：**

**io-threads-do-reads: 配置为yes则开启多线程**

**io-threads: 线程数量，官方建议4核配成2或者3，8核配置成6**
