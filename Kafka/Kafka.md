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
  

### Kafka核心技术与实战-极客
#### 多线程开发消费者实例
##### 知识点
* kafka java consumer 用户主线程 和 心跳线程
* 主线程：启动应用程序main方法的线程，
* 心跳线程：负责定期给对应的Broker机器发送心跳请求，以标识消费者应用的存活性。
* 将心跳线程和主线程的poll方法频率分开，解耦真实的消息处理逻辑与消费者组成员存活性管理；
* kafkaconsumer类不是线程安全的
* 1、每个线程维护专属的consumer实例，负责完整的消息获取，消息处理流程
* 2、消费者使用单或者多线程获取消息，同时创建多个消费线程执行消息处理逻辑，处理消息由特定的线程池来做。
* 多线程方案和多进程方案的优劣：
* 方案2如何管理和提交位移

#### 如何避免消费者组重平衡
##### 知识点
* 重平衡过程中，所有消费者实例共同参与，在协调者组件的帮助下，完成订阅主题分区的分配，整个过程中，所有实例不消费任何消息
* 组成员数量发生变化、订阅主题数量发生变化、订阅主题分区发生变化都会导致重平衡
* 未能及时发送心跳请求，导致consumer被提出group会导致重平衡
* consumer消费时间过长，导致规定的时间内无法消费完所有poll的消息，consumer主动发起离开组的请求，开始重平衡
* 


#### kafka一般什么情况下会丢失消息


#### kafka的副本机制
##### 知识点
* 提供数据冗余
* kafka没有提供高伸缩性、改善数据局部性：因为kafka副本机制追随者副本不对外提供服务
* 在分区层级下定义，每个分区配置若干个副本
* 如何确保所有副本数据一致
  * 采用基于领导者的副本机制
  * 追随者副本不对外提供服务，只从领导者副本异步拉取消息
  * 当leader 挂掉，开启领导者选举
  * 这种副本机制的好处
    * 方便实现read_your_writes,可以马上读取刚才写入的消息
    * 方便实现单调读：不会看到某条消息一会存在一会不存在
  * ISR副本集合：不只是追随者副本集合，也包括leader副本
  * 进入ISR副本集合必须具备条件：follower副本落后leader时间不能太久，默认值是10s。
  * 领导者选举：从ISR集合中选取，如果ISR为空，则从非同步副本中选取（即不在ISR中的存活副本），担可能出现数据丢失。可以提高可用性。
  * LEO 和 HW 机制

#### 请求是怎么被处理的
##### 知识点
* 所有请求都是通过TCP以Socket的方式进行通讯
* KAFKA采用Reactor模式处理请求，适用于处理多个客户端并发向服务器端发送请求的场景
* 每台broker启动时会创建多个网络线程，num.network.threads，专门处理客户端发送的请求
* 网络线程拿到请求，会将请求放到共享请求队列中。Broker端会有一个IO线程池，负责从队列中去除请求，执行真正的处理
* 请求队列是所有网络线程共享，响应队列是每个网络线程专属
* 缓存延时请求：处理一时为满足条件不能立刻处理的请求
* 分为数据类请求和控制类请求，一视同仁（有问题）2.3版本将两类请求分离，实现原理：创建两套网络线程池和IO线程池的组合，分贝处理数据类请求和控制类请求


#### 消费者组重平衡
##### 知识点
* 需要借助coordinator组件
* 组成员数量发生变化，订阅主题数量发生变化，订阅主题分区数发生变化
* 靠消费者的心跳线程通知到其他消费者实例
* 消费者组状态机帮助协调者完成整个重平衡流程
* 消费者组具有5个状态
* 消费者端重平衡解析
  * JoinGroup请求 和  SyncGroup请求
  * SyncGroup 请求的主要目的，就是让协调者把领导者制定的分配方案下发给各个组内成员
* Broker端重平衡场景剖析
  * 新成员入组
  * 组成员主动离组
  * 组成员崩溃离组
  * 组成员提交位移


#### KAFKA控制器
##### 知识点
* 控制器组件主要作用是在zookeeper的帮助下管理和协调整个kafka集群
* 任意一台broker都能充当控制器的角色
* 第一个创建/controller节点的Broker被指定为控制器
* 职责：
  * 主题管理
  * 分区重分配
  * Preferred领导者选举
  * 集群成员管理
  * 提供数据服务，保存了最全的元数据信息（所有主题信息，所有Broker信息，所有涉及运维任务的分区）
* Failover功能：运行中的控制器意外终止，立即弃用备用控制器来代替失败的控制器，无需手动干预


#### 高水位
##### 知识点
* 水位：单调增加且表征最早未完成工作的时间戳，kafka水位和位置信息绑定
* 高水位的作用
  * 定义消息可见性，标识分区下的哪些消息是可以被消费者消费的
  * 帮助kafka完成副本同步
* 高水位：HW   
* 日志末端位移：LEO，指向吓一跳消息到来时的位移
* kafka所有副本都有HW 和LEO
* 在leader副本所在的broker上保存了所有副本的LEO值
* 远程副本的作用：帮助Leader副本更新其高水位值
* 更新时机：
   * Follower副本LEO值更新：Follower副本从Leader副本拉取消息，写入本次磁盘
   * Leader副本LEO值更新：Leader副本接受生产者发送的消息，写入本次磁盘
   * 远程副本LEO值：F 从L拉取消息时，告诉L从哪个位移开始拉取，L使用这个位移值更新远程副本LEO
   * Follower副本HW更新：比较LEO和从Leader副本发来的HW，取较小值更新
   * Leader副本更新HW：取Leader副本和满足条件的所有远程副本中LEO最小值
 * 上述满足条件主要指的是：
   * 副本在ISR中
   * 副本LEO值落后于leader LEO值不超过一定时间
 * Leader副本HW更新时机：
   *  副本成为leader副本时
   *  broker崩溃导致副本被踢出ISR时
   *  producer向副本写入消息时
   *  leader副本处理follower fetch请求时
* 此种方式会导致数据丢失或数据不一致： 根本原因是HW被用来作为衡量副本备份是否成功与否以及出现failture时作为日志截断的依据，但是HW更新是异步延迟的，特别需要额外的fetch请求处理才能更新，这中间发生的任何崩溃都有可能导致HW的过期
#### Leader Epoch（没明白）
##### 知识点
* 主要弥补高水位机制的缺陷
* Epoch：一个单调增加的版本号，没放副本领导权发生变更时，会增加版本号
* 起始位移：Leader副本在该Epoch上写入的首条消息的位移
* 由于L和F 高水位更新是有时间错配的，如果错配时间内接连发生宕机，就会出现数据丢失的情况
* Leader端开辟内存区保存leader的epoch信息
* （epoch，offset）epoch代表leader的版本号，leader变更就会+ 1，offset对应于该epoch版本的leader写入第一条消息的位移


#### kafka主题管理
* 提供kafka-topics脚本，帮助用户创建主题
* 2.2版本之后 使用--bootstrap-server
* --zookeeper 会绕过kafka的安全体系

#### kafka动态配置
* 生产环境中服务器不能随意重启，因此需要动态参数，无需重启即可生效，broker会监听。
* 配置分类
   * read-only：只有重启broker才会生效
   * per-broker:动态参数，只会在对应broker生效
   * cluster-wide：动态参数，在整个集群范围内生效
* 应用场景
  * 动态调整broker端各种线程池大小，实时应对突发流量
  * 动态调整broker端连接信息或安全配置信息
  * 动态更新SSL keystore 有效期
  * 动态调整 broker端compact操作性能
  * 实时变更JMX指标收集器
* 动态参数保存在zookeeper中
* 优先级：per-broker > cluster-wide > static > kafka默认值
* 只能通过kakfa-config脚本来调整动态参数

#### kafka重设消费者组位移
* 重设位移：位移维度，时间维度
* 位移维度：
  * earlist
  * latest：LEO
  * current： 最新已提交
  * specified-offset
  * shift-by-N
* 时间维度：
  * datetime
  * duration
* 两种方法：API 和 kafka-consumer-groups命令行脚本
* API要禁止自动提交位移，调用带长整型的poll方法

#### 常见工具脚本汇总
* 生产消息使用 kafka-console-producer
* 数据消费使用 kafka-console-consumer
* 测试生产者的脚本：kafka-producer-perf-test
* 测试消费者的脚本：kafka-consumer-perf-test
* 查看具体文件的内容：kafka-dump-log
* 查看消费者组位移:kafka-consumer-groups --describe

#### 授权
* ACL： 用户----权限
* RBAC：用户----角色----权限
* 

#### KAFKA 跨集群配置（Mirror Maker）
* 消费者+生产者的程序
```
$ bin/kafka-mirror-maker.sh --consumer.config ./config/consumer.properties --producer.config ./config/producer.properties --num.streams 8 --whitelist ".*"
```
* 部分公司开发了自己的集群镜像工具


#### Kafka监控
* 主机监控：监控kafka集群broker节点所在机器的性能
  * 机器负载，CPU使用率，内存使用率，磁盘IO，网络IO，TCP连接，打开文件数，inode使用情况
* JVM监控：Full GC 发生频率和时长，活跃对象大小，应用线程总数
* 集群监控：
  * 查看Broker进程是否启动，端口是否建立
  * 查看Borker端关键日志
  * 查看Broker端关键线程的运行状态，比如log compaction线程，副本拉取消息的线程
  * 查看关键JMX指标
  * 监控kafka客户端
* 主流监控框架：
  * JMXTool
  * Kafka Manager
  * Burrow
  * JMXTrans + InfluxDB + Grafana


#### KAFKA 优化
* 性能指标：吞吐量 和 延时
* 吞吐量：TPS，指broker端进程或client端应用程序每秒能处理的字节数或消息数
* 延时：表示从Producer端发送消息到Borker端持久化完成之间的时间间隔。
* 优化漏斗：
  * 应用程序层
    * 不要频繁创建Producer 和 consumer对象实例
    * 用完及时关闭，
    * 合理利用多线程改善性能，kafka producer是线程安全的，consumer 不是线程安全的
  * 框架层：合理设置参数
  * JVM层
  * 操作系统层
* 如何从吞吐量和延时方面调优，从broker端，producer端，consumer端分析

