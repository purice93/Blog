---
title: hadoop学习记录1-基本原理 
tags: Hadoop,MapReduce,HDFS
grammar_cjkRuby: true
---

>创始人：Doug cutting有两个有名的开源项目一个是搜索索引器Lucene，之后为了解决Lucene大规模数据问题，创建了Hadoop开源框架。其中Lucene是他妻子的名字，Hadoop是他儿子玩具大象的名字。

* Hadoop简介：两个重要的组成部分，存储和计算
	* 分布式存储系统HDFS （Hadoop Distributed File System ）
		• 分布式存储系统
		• 提供了 高可靠性、高扩展性和高吞吐率的数据存储服务
	* 分布式计算框架MapReduce
		• 分布式计算框架
		•具有 易于编程、高容错性和高扩展性等优点。
* HDFS优点：
	* 高容错性
		•数据自动保存多个副本
		•副本丢失后，自动恢复
	* 适合批处理
		•移动计算而非数据
		•数据位置暴露给计算框架
	* 适合大数据处理
		•GB 、TB 、甚至PB 级数据
		•百万规模以上的文件数量
		•10K+ 节点
	* 可构建在廉价机器上
		•通过多副本提高可靠性
		•提供了容错和恢复机制
* HDFS缺点：这些缺点将是spark的改进部分
	* 低延迟数据访问
		•比如毫秒级
		•低延迟与高吞吐率
	* 小文件存取
		•占用NameNode 大量内存
		•寻道时间超过读取时间
	* 并发写入、文件随机修改
		•一个文件只能有一个写者
		•仅支持append
* HDFS架构：
	![HDFS架构][1] 
* HDFS设计思想:
HDFS 数据存储单元（block）
	* 文件被切分成固定大小的数据块
		•默认数据块大小为64MB ，可配置
		•若文件大小不到64MB ，则单独存成一个block
	* 一个文件存储方式
		•按大小被切分成若干个block ，存储到不同节点上
		•默认情况下每个block都有三个副本
	* Block大小和副本数通过Client端上传文件时设置，文件上传成功后副本数可以变更，Block Size不可变更
* HDFS写流程：
![HDFS写流程][2]
* HDFS读流程：
![HDFS读流程][3]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522655430999.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522655561979.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522655591248.jpg
