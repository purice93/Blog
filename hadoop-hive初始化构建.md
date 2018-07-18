---
title: hadoop-hive初始化 
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>围绕大数据、数据挖掘、人工智能有很多名词，这些名词都互相关联，不太懂的人可能只是把他们当做高级码农的一个分支，但是，其中的真正技术却并不是一半码农能够做的，或者这些领域可能就不是码农干的事，即便做这些的人可能是个java或者python开发者，但是编程只是他们的副业而已，他们的主业却是**数据科学**。但是，有一个问题，既然是副业，也就是说这些数据科学家并不是太会编程，然而，不会编程，数据处理时很艰难的，为了解决这个棘手的问题，程序员们开发了一种新的模式，数据科学家往往都很精通数据库，即sql处理，因此搭建了一个sql到编程的一个桥梁，即hive。

* hive的设计目的
hive设计的目的就是让精通sql技能的分析师能够对利用大数据算法进行数据操作（增删改查），这些大数据算法都是程序员们的做的，或者说是程序员们设计的。

* 更加优美的解释
hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的sql查询功能，可以将sql语句转换为MapReduce任务进行运行。 其优点是学习成本低，可以通过类SQL语句快速实现简单的MapReduce统计，不必开发专门的MapReduce应用，十分适合数据仓库的统计分析。

* hive架构
![enter description here][1]
	* metastore，hive的数据仓库，相当于数据库，对应于hdfs的文件
	* CLI、JDBC、WEB GUI分别对应命令行接口、开发jdbc接口、浏览器接口（解对数据进行查看和操作的窗口）
	* Driver：即驱动器连接器，连接操作到hdfs

* hive搭建
	* 一个hadoop集群错误的处理
	今天启动hadoop集群后，本来是两个namenode，最后active启动了，standby失败了，通过查看启动日志，发现错误总是指向历史任务job，即hadoop集群重启之后总是执行上次失败的任务，导致有缓存存在，无法跳过。错误类似于如下：
	``` stylus
	ERROR org.apache.hadoop.hdfs.server.namenode.FSEditLogLoader: Encountered exception on operation AddOp [length=0, inodeId=16828, path=/usr/output/friendSort/_SUCCESS, replication=3, mtime=1525359058429
	```
	网上的解决办法：[Unable to restrat standby Namenode][2]
	[更加详细的找错步骤，但是解决方式太麻烦了][3]
	``` stylus
	主节点命令输入：启安全模式，并保存，离开
	sudo -u hdfs hdfs dfsadmin -safemode enter
	sudo -u hdfs hdfs dfsadmin -saveNamespace
	sudo -u hdfs hdfs dfsadmin -safemode leave
	
	辅节点命令输入：获取上面保存的fsimage，恢复（具体啥意思我也没弄清楚，应该技术恢复之前的数据，擦除当前的）
	sudo -u hdfs hdfs namenode -bootstrapStandby -force
	```
	* 直接上传hive压缩包，解压，添加环境变量
	* hive配置
	hive配置有三种方式：第一种是本地配置，第二种是使用单机mysql作为仓库，第三种是使用远程mysql数据库。这里使用本地mysql数据库
		* 复制修改配置文件：hive-site.xml，删除所有内部配置信息，添加如下：
		``` stylus
		<property>
  <name>hive.metastore.warehouse.dir</name>
  <value>/user/hive_remote/warehouse</value>
</property>

<property>
　　<name>javax.jdo.option.ConnectionURL</name>
　　<value>jdbc:mysql://localhost:3306/metastore?createDatabaseIfNotExist=true</value>
</property>
<property>
　　<name>javax.jdo.option.ConnectionDriverName</name>
　　<value>com.mysql.jdbc.Driver</value>
</property>
<property>
　　<name>javax.jdo.option.ConnectionUserName</name>
　　<value>root</value>
</property>
<property>
　　<name>javax.jdo.option.ConnectionPassword</name>
　　<value>root</value>
</property>
		```
		* 安装mysql，并创建metastore数据库，设置用户权限（否则无法访问）[安装连接][4]
		* 启动hive，成功：直接输入hive命令（这里面错误可能很多，部分错误解决办法如下）

* 错误记录：（20%的时间学习，80%的时间在找错）
	* java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
	启动metastore `hive --service metastore `
	* Could not create ServerSocket on address 0.0.0.0/0.0.0.0:9083.
	hive metastore进程 已经有启动过了，应该讲相关的进程kill 掉。`ps -ef |grep  hive `将hive 相关进程kill 掉。`kill -9 2545`,然后重新启动：`hive --service metastore`
	* 启动hive时报错Access denied for user 'root'@'hadoop01' (using password: YES)
	原因：权限问题或者密码问题
	密码问题去看配置，权限问题：第一步：查看mysql数据库所有的权限数据`select host,user,password from mysql.user;`，修改权限`update mysql.user set host = '%' where user = 'root' and host = '127.0.0.1';`，刷新修改`flushprivileges;`[参照博客][5]
	* hive 安装警告 WARN conf.HiveConf: HiveConf of name hive.metastore.local does not exist
	在0.10  0.11或者之后的HIVE版本 hive.metastore.local 属性不再使用。在配置文件hive-site.xml 中删除hive.metastore.local配置项
	* mysql unrecognized service问题解决
	centos安装mysql公有三个服务，必须需都安装完全。通过`rpm -q mysql`查看mysql安装的服务，少那个重新安装
	* hive无命令
	环境变量没生效：[配置环境变量三种方法，选择第二种][6]


*更多hive内容：*[Hive 接口介绍（Web UI/JDBC）配置][7]

相关博客：
[安装讲解更加详细，细到环境变量][8]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525617300292.jpg
  [2]: https://community.hortonworks.com/questions/34636/unable-to-restrat-standby-namenode.html
  [3]: http://ju.outofmemory.cn/entry/98888
  [4]: http://www.runoob.com/mysql/mysql-install.html
  [5]: https://blog.csdn.net/anmo1221/article/details/79145973
  [6]: https://blog.csdn.net/huangfei711/article/details/53044539
  [7]: https://hexo2hexo.github.io/Hive%20%E6%8E%A5%E5%8F%A3%E4%BB%8B%E7%BB%8D/
  [8]: http://www.cnblogs.com/edisonchou/p/4426096.html
