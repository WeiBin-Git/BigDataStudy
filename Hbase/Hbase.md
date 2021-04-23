#### HDFS
* 分布式存储工作机制：
  * 元数据存储在NameNode
  * 数据块存储在DateNode
  * 客户端与NameNode交互，实现读写流程
  * 读：客户端发送读写请求，NameNode判断客户端是否有权限，文件是否存在，将block所在datanode节点发送到客户端，客户端拿到位置去datanode读，读取之前进行checksum操作判断校验和是否正确，读完后再向Namenode发送请求询问是否还有block块，没有的话客户端执行close方法，将读取到的文件合并成一个大文件
  * 写：客户端携带路径向NameNode请求，NameNode进行判断，无误返回可以写入，客户端进行切片，上传第一个block块，namenode根据存储空间，机架感知原理等判断存储在哪个datanode，客户端回去datanode建立管道，然后开始写入，一次进行流水线复制，都写入完成后会将写入完成的信息返回给namenode，namenode存储各个block块的元数据信息
* 小文件问题及解决方案
  * 所有文件存储在namenode
  * 文件元数据信息在namenode平均150字节
  * 影响MR任务的数量，MR会对一个小文件开启一个MapTask，MapTask过多会降低执行效率；
  * hadoop2.0 默认块大小128Mb：
  * 访问小文件严重影响性能
  * 自带了几种解决方案：HAR , Sequence File， CombineFileInputFormat
* 高可用原理
  * 运行两个NameNode，一个active，一个standby状态
  * active负责数据的读写，standby负责同步操作
  * 预防脑裂有是三种隔离机制：共享存储隔离；客户端隔离；datenode隔离；