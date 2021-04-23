#### kafka
* kafka架构
  * producer
  * consumer
  * zookeeper：存储消费者与分区之间的对应关系，brokerID，消费者offset
  * kafka集群：broker为单位，broker中有topic，topic中有partition，分区内有segment，segment里有.index文件和.log文件
* 框架原理：分布式，易于扩展；同时为发布和订阅提供高吞吐量；支持多订阅者，自动重平衡；消息持久化
* 如何避免丢失数据
 *  生产者：ack=0
 *  broker：通过副本机制
 *  消费者：通过offset机制，关闭自动提交offset，消息完整处理完，再手动提交
* kafka性能好的原因
    * partition顺序读写，充分利用磁盘特性，这是基础；
    * Producer生产的数据持久化到broker，采用mmap文件映射，实现顺序的快速写入；
    * Customer从broker读取数据，采用sendfile，将磁盘文件读到OS内核缓冲区后，直接转到socket buffer进行网络发送。
    mmap 和 sendfile总结
    * 都是Linux内核提供、实现零拷贝的API；
    * sendfile 是将读到内核空间的数据，转到socket buffer，进行网络发送；
    * mmap将磁盘文件映射到内存，支持读和写，对内存的操作会反映在磁盘文件上。RocketMQ 在消费消息时，使用了 mmap。kafka 使用了 sendFile。
* 零拷贝
  * 传统流程： 磁盘文件--操作系统内核缓冲区--应用程序缓存--socket网络发送缓冲区-- 网卡发送  经过四次拷贝
  * 第二个和第三个不需要，可以将数据从操作系统缓冲区到套接字缓冲区
* 分了几个区，为什么分，分区数小于broker数量
* kafka消费策略：
  * 一个消费组的一个消费者同时只能消费一个topic中一个分区的一条数据
  * 消费者将消费的offset保存到zk中，可以保证数据消费精准
  * 提交offset方式有两种：自动提交，手动提交，手动提交可以避免数据不更新导致重复消费的问题
* 消息传递方式：同步发送；发送并忘记，配合ack=0使用；异步+回调ack=-1；
* 消费一致性问题：不管是新的leader还是新选区的leader，consumer读取到的数据一致；当老的leader挂了，consumer从最小的offset去消费
* offset提交的方式：手动提交；异步手动提交+回调函数；异步+同步
* 多个分区多个消费者读取数据不是有序的
* 优化
  * broker ：优化处理消息最大线程数；处理磁盘IO线程数；调整log文件刷盘策略；日志保存策略；副本机制调成2个；
  * producer：调整未发送出去消息的缓冲区大小；默认不压缩，配置合适的压缩方式；
  * consumer：增加consumer数量，提高并发度；
  * 内存：默认一个G,建议不超过6个G
* 幂等性
  * 消息重复的原因：客户端发送消息网络错误导致重试；
* ISR
* 消费者组状态：empty、dead、preparingReblance、completingReblance、stable
* 重平衡
  * 重平衡是什么？本质上是一种协议，规定了一个消费者组下所有消费者如何达成一致来分配订阅每个分区
  * 重平衡发生时机？1：订阅主题数发生变化；2：主题分区发生变化；3：消费者组成员发生变化（处理消息超时，心跳超时）