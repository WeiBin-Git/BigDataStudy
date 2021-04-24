
#### DataX
##### 基本概念
* 异构数据源离线同步工具，实现关系型数据库、HDFS、Hive、ODPS、Hbase、FTP之间的数据同步
* 单机，但是效率块，通过调度系统实现分布式功能
* 框架设计:ReadPlugin --> FrameWork --> WritePlugin
* 运行原理
  * Job：负责数据清理，任务划分为Task，交给调度器，TaskGroup管理
  * Task：由Job切分开来，作业的最小单位，每个Task负责一部分数据同步工作
  * Scheduler： 将Task组成TaskGroup，单个TaskGroup的并发数量是5
  * TaskGroup： 描述的是一组Task集合。在同一个TaskGroupContainer执行下的Task集合称之为TaskGroup
* 运行模式
  * Standalone
  * Local
  * Distributed
* DataX二次开发
  * https://github.com/alibaba/DataX/blob/master/README.md
  * https://github.com/alibaba/DataX/blob/master/dataxPluginDev.md
  
#### Sqoop
##### 基本概念
* Hadoop 和传统数据库之间的数据传输工具
* 运行原理：将导入导出命令翻译成MapReduce程序来实现；
* 在翻译出的Mapreduce 主要是对inputformat 和 outputformat进行定制；


