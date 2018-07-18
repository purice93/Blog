---
title: Hadoop-HBase集群搭建
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>随着数据的增大，传统的关系型数据库对于上千万或者上亿的数据处理，效率会迅速下降。同样是为了解决大数据处理问题，hbase也是基于分布式，这种依靠列存储的方式，使得数据处于非结构化或者半结构化，便于数据的大量操作。

* hadoop生态架构
![enter description here][1]

* 数据提取工具：
	* flume：收集日志，从日志中提取数据
	* sqoop：从结构化存储器中提取数据

* 数据处理工具：
	* mahout：数据挖掘/机器学习开发库
	* pig：将其他语言转化为mapreduce处理
	* hive：将sql语言转化为mapreduce处理
*核心架构：
	* mapreduce：分布式计算框架
	* hbase：分布式数据库
	* hdfs：分布式文件系统
* zookeeper：分布式协作服务，保证高可用，备份等


* Hbase集群架构
![enter description here][2] 
	* 基础：hadoop集群搭建完成
	* hbase安装包：hbase-1.1.3-bin.tar.gz

* 步骤：参照官方文档5.[hbase官网][3]
	* 解压安装包到指定目录
	* 修改配置：hbase-env.sh
		* 配置java\_home 和 export HBASE\_MANAGES_ZK=false
		* 配置export HBASE_CLASSPATH=/home/hadoop-2.5.1/etc/hadoop/，这里是hadoop配置文件的路径
	* 配置hbase-site.xml
	```
	<property>
	<name>hbase.rootdir</name>
	<value>hdfs://node1:8020/hbase</value>
	</property>
	<property>
	<name>hbase.cluster.distributed</name>
	<value>true</value>
	</property>
	<property>
	<name>hbase.zookeeper.quorum</name>
	<value>node1,node2,node3</value>
	</property>

	```
	* 配置reginservers，数据节点：修改文件reginservers
	``` stylus
	node1
	node2
	node3
	```
	* 配置完成，复制到node2,node3,使所有环境变量生效
	* 启动：start-hbase.sh
![enter description here][4]

* phonenix安装
>由于hbase自身对一些功能不支持，所以，通过phonenix来实现。具体为：phoenix最主要给HBase添加了二级索引、SQL的支持。

	* 解压相应的包 ，比如phoenix-4.5.2-HBase-1.1-bin.tar.gz
	* 将解压后的包里的phoenix-core-4.5.2-HBase-1.1.jar拷贝到集群各个节点HBase的lib目录下。这里的包看版本了，记住前缀是phoenix-core的包，如果这里有phonenix旧的包需要先删掉
	* 重启hbase集群
	* bin/sqlline.py node1:2181,如下，表示成功
![enter description here][5]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526030685422.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526031283775.jpg
  [3]: http://hbase.apache.org/book.html#standalone_dist
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526031983177.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526035360760.jpg
