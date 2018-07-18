---
title: hadoop学习记录2-Hadoop安装配置
tags: Hadoop,MapReduce,HDFS
grammar_cjkRuby: true
---

* 安装前的环境：四台机器的时间一致；需要一台机器进行免密码登录，即可以访问任何一台机器，包括自己，而不需要输入密码。这样便于通过一台机器进行控制，而且避免每一次都需要输入密码。
	* 时间一致：
		* `date`查看时间
		* `ntpdate -u xx.xx.xx.xx`同步xx.xx.xx.xx服务器的时间到本机，一般使用`ntpdate -u ntp.api.bz`。[参看链接][1]
		![ntpdate][2]
	* 设置免密码登录Setup passphraseless ssh[参考Hadoop][3]
		* ssh登录方式：`ssh 192.168.116.3`
		* 第一步：node1上生成秘钥文件，公共秘钥和私有秘钥`ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa`
		* 第二步，将公共秘钥加入到认证文件中：`cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys`。到此node1能够访问node1自己了。
		* 第三部，将node1的公钥发送给node234，并添加到node234的认证文件中。node1中执行`scp ~/.ssh/id_dsa.pub root@node2:/opt/`，node2中执行`ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa`加上`cat /opt/id_dsa.pub >> ~/.ssh/authorized_keys`。其他node3、4同理。
		![过程][4]
		![密匙验证原理][5]
		* 第四步，解压Hadoop到linux指定文件，编辑`etc/hadoop/hadoop-env.sh`修改参数`export JAVA_HOME=/usr/java/jdk1.7.0_79/`。
		* 修改hadoop/core-site.xml文件、hadoop/hdfs-site.xml文件、slaves文件，添加masters文件
		![core-site.xml][6]
		![hdfs-site.xml][7]
		![slaves][8]
		![masters][9]
		![结构配置][10]
		* 配置完成。将node1的配置复制到另外三个`scp -r hadoop-2.5.1/ root@node2:/home/`
		* 配置hadoop环境变量`vi ~/.bash_profile`。复制到另外三个。`source ~/.bash_profile`生效
		![hadoop环境变量][11]
	* 启动
		* 1、格式化hdfs`bin/hdfs namenode -format`
		* 2、启动NameNode daemon and DataNode daemon`start-dfs.sh`
		![启动界面][12]
	* 访问验证
	![namenode和secondarynode][13]


  [1]: https://www.linuxprobe.com/linux-time-synchronization.html
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522656807914.jpg
  [3]: http://hadoop.apache.org/docs/r2.5.2/hadoop-project-dist/hadoop-common/SingleCluster.html
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522657594965.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522658748946.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522661067782.jpg
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522661428174.jpg
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522662021254.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522662109542.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522661793031.jpg
  [11]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522662805110.jpg
  [12]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522663594531.jpg
  [13]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522664133432.jpg
