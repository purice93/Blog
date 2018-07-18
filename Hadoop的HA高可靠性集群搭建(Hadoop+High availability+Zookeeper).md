---
title: Hadoop的HA高可靠性集群搭建(Hadoop+High availability+Zookeeper)
tags: High availability,Hadoop,Zookeeper
grammar_cjkRuby: true
---

一.概述
部分转载自：http://eksliang.iteye.com/blog/2226986

* 1.1 hadoop1.0的单点问题
Hadoop中的NameNode好比是人的心脏，非常重要，绝对不可以停止工作。在hadoop1时代，只有一个NameNode。如果该NameNode数据丢失或者不能工作，那么整个集群就不能恢复了。这是hadoop1中的单点问题，也是hadoop1不可靠的表现。如下图所示，便是hadoop1.0的架构图；
![enter description here][1]

* 1.2 hadoop2.0对hadoop1.0单点问题的解决
为了解决hadoop1中的单点问题，在hadoop2中新的NameNode不再是只有一个，可以有多个（目前只支持2个）。每一个都有相同的职能。一个是active状态的，一个是standby状态的。当集群运行时，只有active状态的NameNode是正常工作的，standby状态的NameNode是处于待命状态的，时刻同步active状态NameNode的数据。一旦active状态的NameNode不能工作，通过手工或者自动切换，standby状态的NameNode就可以转变为active状态的，就可以继续工作了。这就是高可靠。

* 1.3 使用JournalNode实现NameNode(Active和Standby)数据的共享
Hadoop2.0中，2个NameNode的数据其实是实时共享的。新HDFS采用了一种共享机制，Quorum Journal Node（JournalNode）集群或者Nnetwork File System（NFS）进行共享。NFS是操作系统层面的，JournalNode是hadoop层面的，我们这里使用JournalNode集群进行数据共享（这也是主流的做法）。如下图所示，便是JournalNode的架构图。
![enter description here][2]

 两个NameNode为了数据同步，会通过一组称作JournalNodes的独立进程进行相互通信。当active状态的NameNode的命名空间有任何修改时，会告知大部分的JournalNodes进程。standby状态的NameNode有能力读取JNs中的变更信息，并且一直监控edit log的变化，把变化应用于自己的命名空间。standby可以确保在集群出错时，命名空间状态已经完全同步了

* 1.4 NameNode之间的故障切换
对于HA集群而言，确保同一时刻只有一个NameNode处于active状态是至关重要的。否则，两个NameNode的数据状态就会产生分歧，可能丢失数据，或者产生错误的结果。为了保证这点，这就需要利用使用ZooKeeper了。首先HDFS集群中的两个NameNode都在ZooKeeper中注册，当active状态的NameNode出故障时，ZooKeeper能检测到这种情况，它就会自动把standby状态的NameNode切换为active状态。

二.Hadoop（HDFS式 HA）集群的搭建[参考文档][3]
* 架构图
![架构图][4]
* 2.1 配置详细
四台机器：ndoe1，ndoe2，ndoe3，node4
![配置][5]
node1:192.168.116.3
node2:192.168.116.4
node3:192.168.116.5
node4:192.168.116.6

* 搭建zookeeper集群，[参考链接][6]

* HDFS High Availability
参考文档路径：hadoop-2.5.2/share/doc/hadoop/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithNFS.html

* 配置详解
	
1、 修改hadoop文件hdfs-site.xml

``` stylus
<configuration>
    <property>
        <name>dfs.nameservices</name>
        <value>bjsxt</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.bjsxt</name>
        <value>nn1,nn2</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.bjsxt.nn1</name>
        <value>node1:8020</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.bjsxt.nn2</name>
        <value>node2:8020</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.bjsxt.nn1</name>
        <value>node1:50070</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.bjsxt.nn2</name>
        <value>node2:50070</value>
    </property>
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://node2:8485;node3:8485;node4:8485/bjsxt</value>
    </property>
    <property>
        <name>dfs.client.failover.proxy.provider.bjsxt</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
<property>
  <name>dfs.ha.fencing.methods</name>
  <value>sshfence</value>
</property>
<property>
  <name>dfs.ha.fencing.ssh.private-key-files</name>
  <value>/root/.ssh/id_dsa</value>
</property>
<property>
  <name>dfs.journalnode.edits.dir</name>
  <value>/opt/hadoop-2.5/data</value>
</property>
<property>
   <name>dfs.ha.automatic-failover.enabled</name>
   <value>true</value>
</property>
</configuration>
```

2、core-site.xml

``` stylus
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://bjsxt</value>
    </property>


    <property>
        <name>ha.zookeeper.quorum</name>
        <value>node1:2181,node2:2181,node3:2181</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/hadoop-2.5</value>
    </property>
</configuration>
```
3、将当前node下配置好的hadoop复制到其他node2\3\4

``` stylus
scp /home/hadoop-2.5.1/etc/hadoop/ root@node2:/home/hadoop-2.5.1/etc/
scp /home/hadoop-2.5.1/etc/hadoop/ root@node3:/home/hadoop-2.5.1/etc/
scp /home/hadoop-2.5.1/etc/hadoop/ root@node4:/home/hadoop-2.5.1/etc/
```
4、启动journalnode，即234（单节点启动）
在node2、3、4中分别输入以下命令
``` stylus
hadoop-daemon.sh start journalnode
```
5、检查：
为了防止出错，在每一步修改完后需要通过查看日志logs来进行检查，如果没有异常，则正确，继续进行
journalnode和namenode的检查：
``` stylus
tail -100 /home/hadoop-2.5.1/logs/hadoop-root-journalnode-node3.log
tail -100 /home/hadoop-2.5.1/logs/hadoop-root-namenode-node3.log 
```
6、同步两个namenode的数据，即node1和node2,
为了同步数据，首先需要选择一个namenode进行数据格式化

``` stylus
hdfs namenode -format
```
格式化后，将当前节点node1的数据拷贝到备用节点node2下

``` stylus
scp -r /opt/hadoop-2.5 root@node2:/opt/
```


7、初始化zookeeper中的HA状态Initializing HA state in ZooKeeper
选择一个namenode输入如下命令：

``` stylus
hdfs zkfc -formatZK
```
输入`start-dfs.sh`启动集群

8、验证
![访问是否正常][7]
![验证是否正常切换][8]（如果无法正常切换，查看日志，可能是node节点之间无法免密码登录）


* 添加ResourceManager用于资源管理
1、官方架构
![架构][9]
2、配置ResourceManager节点
node1和node2作为ResourceManager节点
3、修改yarn-default.xml
``` stylus
<property>
   <name>yarn.resourcemanager.ha.enabled</name>
   <value>true</value>
 </property>
 <property>
   <name>yarn.resourcemanager.cluster-id</name>
   <value>cluster1</value>
 </property>
 <property>
   <name>yarn.resourcemanager.ha.rm-ids</name>
   <value>rm1,rm2</value>
 </property>
 <property>
   <name>yarn.resourcemanager.hostname.rm1</name>
   <value>node1</value>
 </property>
 <property>
   <name>yarn.resourcemanager.hostname.rm2</name>
   <value>node2</value>
 </property>
 <property>
   <name>yarn.resourcemanager.zk-address</name>
   <value>node2:2181,node3:2181,node4:2181</value>
 </property>
```
4、单节点配置

``` stylus
etc/hadoop/core-site.xml:

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
etc/hadoop/hdfs-site.xml:

<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```
将配置文件复制到其他节点上

``` stylus
scp /home/hadoop-2.5.1/etc/hadoop/ root@node2:/home/hadoop-2.5.1/etc/
scp /home/hadoop-2.5.1/etc/hadoop/ root@node2:/home/hadoop-2.5.1/etc/
scp /home/hadoop-2.5.1/etc/hadoop/ root@node2:/home/hadoop-2.5.1/etc/
```
5、启动yarn

``` stylus
1、主节点node1启动：
start-yarn.sh
2、启动备用yarn（node2上）：
yarn-daemon.sh start resourcemanager 
```
6、验证resourcemanager
![cluster][10]



**错误记录**
1、防火墙关闭
2、node之间的免密码登录
3、配置文件的单词错误以及单词一致性
4、查日志




可能更好的博客连接：
[https://blog.csdn.net/q361239731/article/details/53559681][11]
[https://blog.csdn.net/q361239731/article/details/53559866][12]
    





 


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522722123247.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522722149664.jpg
  [3]: https://zookeeper.apache.org/doc/r3.4.6/zookeeperStarted.html
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524749982782.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524749742247.jpg
  [6]: https://blog.csdn.net/soundslow/article/details/79781456
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524751476540.jpg
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524751714467.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524751962003.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524752843479.jpg
  [11]: https://blog.csdn.net/q361239731/article/details/53559681
  [12]: https://blog.csdn.net/q361239731/article/details/53559866
