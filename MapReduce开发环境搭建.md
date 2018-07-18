---
title: MapReduce开发环境搭建
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>MapReduce运行在linux系统上，但是对于开发而言，系统平台可能是windows平台，因为在看法测试时，需要让程序运行在Linux上的hadoop集群上。第一种方式是：建立windows和linux之间的连接，开发平台eclipse在windows上，调用hadoop的运行接口和文件存储接口。（windows上没有配置hadoop，因此无法直接在windows上调试，只能开发完，打包之后发送到linux上执行）。第二种方式：windows上配置hadoop，在eclipse中连接linux进行运行测试（无法调试）。（真正企业运行情况）。第三种：可以调试。配置本地hadoop文件。前两种也叫作服务器环境，第三种为本地测试环境

* 本地测试环境
	1. 将hadoop解压拷贝到windows系统中，在bin目录下添加winutils.exe
	2. 配置hadoop环境变量
	3. 修改hadoop源码，即覆盖hadoop自带的jar包
	![enter description here][1]
	4. 配置linux上的hadoop连接
	main函数中加入
	``` stylus
	Configuration config = new Configuration();
	config.set("fs.defaultFS", "hdfs://node1:8020");
	config.set("yarn.resourcemanager.hostname", "node1");
	```
	![enter description here][2]
	
* 服务器环境
一、直接打包jar，上传到linux服务器，运行`hadoop jar filename.jar com.package.MainRun`
二、eclipse中直接调用，执行过程还是在linux上
	1. 打包为jar文件，直接放在本地
	2. 同上修改hadoop源码
	3. 增加一个属性`config.set("mapred.jar", "C:\\Users\\Administrator\\Desktop\\wc.jar");`
	![enter description here][3]
	4. 将服务器上的hadoop配置文件复制到本地工程的src下
	![enter description here][4]

结果：
![enter description here][5]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525258516410.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525259125868.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525259385008.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525259322734.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525259428849.jpg
