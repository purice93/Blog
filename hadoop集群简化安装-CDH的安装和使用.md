---
title: hadoop集群简化安装-CDH的安装和使用
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>hadoop大数据开发环境，由于牵扯到太多的部件，而且这些部件之间联系复杂，独立的安装往往错误百出，即浪费时间又浪费精力，还不讨好，而且这些往往还不是真正开发做的事（可能）。另外对于大数据而言，机器往往动不动就上万台，像这样一台一台的安装，只能把猴子给累死。所以，为了便于继承搭建，hadoop出了一个实用版的CDH用来管理所有的部件，相当于集成。这样就可以慧姐在控制台搭建管理集群，大大解放生产力。

* 系统环境准备
	* 1、克隆-网络配置
	克隆之后的操作系统需要重新分配物理地址
	删除/etc/sysconfig/network-scripts/ifcfg-eth0 文件中的物理地址（删除两行：UUID和物理地址）
	修改IPADDR
	![enter description here][1]	
	删除文件/etc/udev/rules.d/70-persistent-net.rules`rm -rf /etc/udev/rules.d/70-persistent-net.rules`
	分别修改主机名和hosts
	```
	vi /etc/sysconfig/network
	vi /etc/hosts
	```
	重启启动linux: `init 6`
	* 2、SSH免密钥登录（将主节点server公共密匙传给所有的agent）
	```
	ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
	cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
	```
	* 3、防火墙关闭
	```
	service iptables stop
	chkconfig iptables off
	```
	* 4、SELINUX关闭
	``` bash
	setenforce 0
	vi /etc/selinux/config (SELINUX=disabled)
	```
	* 5、安装JDK配置环境变量
	* 6、安装NTP，进行时间同步
	设置开机启动`chkconfig ntpd on`
	设置时间同步`ntpdate -u time.nist.gov`
	这里可能需要先命令行安装ntpdate`yum install -y ntp`
	* 7、安装配置mysql5.5：[命令安装模式参照][2]
	
	* 第一步就是看linu是否安装了mysql`rpm -qa|grep mysql`
	* 第二步，卸载mysql5.1`rpm -e mysql-libs --nodeps`
	* 第三步，yum中之后mysql5.1，安装还是5.1，需要增加repo：
	```
	yum install http://dev.mysql.com/get/mysql-community-release-el6-5.noarch.rpm
	```
	* 第四步：调整为yum安装时的MySQL版本为5.5。（可能为5.6，所以需要调整）
	```
	yum-config-manager --disable mysql56-community
	yum-config-manager --enable mysql55-community
	```
	当显示以下 yum-config-manager: command not found,安装相应工具`yum -y install yum-utils`
	* 第五步：安装mysql5.5`yum install mysql mysql-devel mysql-server mysql-utilities`
	![enter description here][3]
	``` sql
	service mysqld start # 启动mysql服务
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
	```
	* 8、下载第三方依赖包(跳过)
	``` bash
	chkconfig、python (2.6 required for CDH 5)、bind-utils、psmisc
	```

* Cloudera Manager安装
	* 1、安装Cloudera Manager Server、Agent(就是解压安装，逻辑对就行，所有节点)
	``` vim
	mkdir /opt/cloudera-manager
	tar xvzf cloudera-manager*.tar.gz -C /opt/cloudera-manager
	```
	* 2、创建用户cloudera-scm
	``` dsconfig
	useradd --system --home=/opt/cloudera-manager/cm-5.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
	```
	* 3、配置CM Agent客户端
	修改文件`vi /opt/cloudera-manager/cm-5.4.3/etc/cloudera-scm-agent/config.ini`中server\_host（服务节点，统一为一个cdh1）以及server_port(默认就行)
	* 4、配置CM Server数据库(只需要在主节点配置)
	拷贝mysql-connector-java-5.1.26-bin.jar文件到目录/usr/share/java/
	注意jar包名称要修改为mysql-connector-java.jar，创建文件夹`mkdir /usr/share/java/`
	
	创建mysql连接权限和登录密码
	``` sql
	grant all on *.* to 'temp'@'%' identified by 'root' with grant option;
	mysqladmin -u root password root# 修改mysql默认登录密码，即最后一个root
	cd /opt/cloudera-manager/cm-5.4.3/share/cmf/schema/
	./scm_prepare_database.sh mysql temp -h cdh1 -uroot -proot --scm-host cdh1 scm scm scm
	```
	如果出错，请重启mysql`service mysqld restart`
	格式：数据库类型、数据库、数据库服务器、用户名、密码、cm server服务器
	![enter description here][4]
	![enter description here][5]
	drop user 'temp'@'%';
	* 5、创建Parcel目录
	一个Server节点
	``` groovy
	mkdir -p /opt/cloudera/parcel-repo
	chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo
	```
	三个Agent节点
	``` groovy
	mkdir -p /opt/cloudera/parcels
	chown cloudera-scm:cloudera-scm /opt/cloudera/parcels
	```
	* 6、制作CDH本地源
	下载好文件CDH-5.4.0-1.cdh5.4.0.p0.27-el6.parcel以及manifest.json，将这两个文件放到server节点的/opt/cloudera/parcel-repo下。
	（这个提前修改好就不用了）打开manifest.json文件，里面是json格式的配置，找到与下载版本相对应的hash码，新建文件，文件名与你的parel包名一致，并加上.sha后缀，将hash码复制到文件中保存。
	* 7、启动CM Server、Agent
	``` groovy
	cd /opt/cloudera-manager/cm-5.4.3/etc/init.d/
	./cloudera-scm-server start
	```
	Sever首次启动会自动创建表以及数据，不要立即关闭或重启，否则需要删除所有表及数据重新安装
	（server节点后台启动，需要10分钟左右，别慌；可以通过查看日志来判断启动进程）
	查看日志`tail -f /opt/cloudera-manager/cm-5.4.3/log/cloudera-scm-server/cloudera-scm-server.log`
	启动成功如下：
	![enter description here][6]
	``` sqf
	./cloudera-scm-agent start
	```
* 访问：http://ManagerHost:7180
用户名、密码：admin
若可以访问，则CM安装成功。
![enter description here][7]

* CDH5安装
	* 指定集群本地主机
	![enter description here][8]
	* 存储库
	![enter description here][9]
	* 集群自动安装，分发到其他节点
	![enter description here][10]
	这里就是CDH的作用，你只需要配置一个服务节点，他就可以自动将配置分发到其他节点
	* 群集设置-选择服务-这里只选择三个
	![enter description here][11]
	![enter description here][12]
	* 数据库设置
	![enter description here][13]
	* 安装完成
	![enter description here][14]


* 最终我卡在了：Host Monitor 启动失败
尝试了很多次，还是失败了，心也累了，先放下吧，毕竟只是一个工具
很多方法都试了，但是都失败了。最后一个原因说是，机子性能不行，可能是这个原因，跑到最后都是卡死了。
错入类似下图
![enter description here][15]



错误记录：
* 1、Could not find a HOST_MONITORING nozzle from SCM  
重启server
* 2、如果界面中没有节点，表明agent节点死了，需要重启
* 3、主节点cdh1无法激活：磁盘空间不足，需要扩展
![enter description here][16]
磁盘扩展方式如下：[正确的磁盘扩展方式][17]
由于安装失败，需要删除部分组件，重新安装：[CDH安装失败了，如何重新安装][18]
* 4、Cloudera 建议将 /proc/sys/vm/swappiness 设置为 0。当前设置为 60。使用 sysctl 命令在运行时更改该设置并编辑 /etc/sysctl.conf 以在重启后保存该设置。您可以继续进行安装，但可能会遇到问题，Cloudera Manager 报告您的主机由于交换运行状况不佳。以下主机受到影响： 
![enter description here][19]
解决办法：在会受到影响的主机上执行echo 0 > /proc/sys/vm/swappiness命令即可解决。(在每一台机器执行)，重新检测
* 5、命令详细信息: 创建 /tmp 目录失败
![enter description here][20]
可能原因：命令超时。解决办法：再次重试，错一次重启一次，直到所有的都完成
* 6、重装HDFS出错
错误描述： 正在检查 NameNode 的名称目录是否为空。仅在为空时格式化 HDFS。
错误原因： 旧的hdfs文件会对新安装HDFS产生干扰。
解决办法：在namenode和所有datanode上执行`rm -rf /dfs`。
* 7、CDH的 monitor内存问题[参考博客][21]
![enter description here][22]
![一步步解决没一个警告][23]

参考链接：
[错误记录][24]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526442284651.jpg
  [2]: https://yunjin-keji.com/install-mysql-55-on-centos
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526473174583.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526477399262.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526477389680.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526387379536.jpg
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526387420216.jpg
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526387667496.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526387828346.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526387914869.jpg
  [11]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526395166530.jpg
  [12]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526526608130.jpg
  [13]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526527030243.jpg
  [14]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526536612591.jpg
  [15]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526567811577.jpg
  [16]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526519067434.jpg
  [17]: https://my.oschina.net/u/876354/blog/967848
  [18]: http://www.cnblogs.com/ivictor/p/4846358.html
  [19]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526525865632.jpg
  [20]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526528519254.jpg
  [21]: https://blog.csdn.net/vbaspdelphi/article/details/53169241
  [22]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526545993434.jpg
  [23]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526546001962.jpg
  [24]: http://www.vicviz.com/loganalyser_install/
