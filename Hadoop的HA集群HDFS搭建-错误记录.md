---
title: Hadoop的HA集群HDFS搭建-错误记录
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

1. Live Nodes显示为0

![正常节点应该显示为3][1]
	* 第一步：查看datanode日志`tail -100 /home/hadoop-2.5.1/logs/hadoop-root-datanode-node2.log`
	* 异常问题：![namenode和datanode的ID不一致][2]
	* 原因：重复格式化。在第一次格式化dfs后，启动并使用了hadoop，后来又重新执行了格式化命令（hdfs namenode -format)，这时namenode的clusterID会重新生成，而datanode的clusterID 保持不变。
	* 解决办法：将name/current下的VERSION中的clusterID复制到data/current下的VERSION中，覆盖掉原来的clusterID，即保持一致。
* 重新启动
	
	参考链接：[https://my.oschina.net/zhongwenhao/blog/603744][3]
2. eclipse上传文件到Hadoop失败的原因及解决方法
![正确上传文件][4]
原因：系统没有权限
解决办法：修改文件`vim hadoop/etc/hadoop/hdfs-site.xml`
添加配置：
``` stylus
<property>
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
```

3. hadoop报错：could only be replicated to 0 nodes, instead of 1
原因：多次格式化hadoop导致版本信息不一致
解决办法：停掉所有服务，重新格式化
	* 先把服务都停掉     stop-all.sh
	* 格式化   hadoop namenode -foramt
	* 重新启动所有服务   start-all.sh
注意，这里可能会导致问题1！！


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524820706336.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524820969980.jpg
  [3]: https://my.oschina.net/zhongwenhao/blog/603744
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524821412207.jpg
