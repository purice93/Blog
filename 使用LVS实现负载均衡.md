---
title: 使用LVS实现负载均衡
tags: lvs,负载均衡，ipvsadm
grammar_cjkRuby: true
---

>为了达到负载均衡，我们需要将上行和下行进行分离（请求和响应进行分离）。具体就是所有的请求都是通过nginx来转发到之后的tomcat服务器；但是，之后的响应数据由对应的tomcat直接发送给客户端，而不需要经过nginx前端服务器。实际上，网络通信中，请求数据总是比响应数据要少得多，请求数据一般只是一个表单的提交，很少涉及到图片或者视频等其他资源；但是服务器响应却存在大量的信息，包括页面渲染等图片和其他各种数据信息。所以，这样就可以达到一个目的：请求数据少，即对请求数据进行集中处理分发；响应数据大，就对响应数据进行分发处理，单独处理。

* Linux服务器集群系统-LVS简介（Linux Virtual Server）(创建人：章文嵩)
>针对高可伸缩、高可用网络服务的需求，我们给出了基于IP层和基于内容请求分发的负载平衡调度解决方法，并在Linux内核中实现了这些方法，将一组服务器构成一个实现可伸缩的、高可用网络服务的虚拟服务器。
>虚拟服务器的体系结构如图2所示，一组服务器通过高速的局域网或者地理分布的广域网相互连接，在它们的前端有一个负载调度器（Load Balancer）。负载调度器能无缝地将网络请求调度到真实服务器上，从而使得服务器集群的结构对客户是透明的，客户访问集群系统提供的网络服务就像访 问一台高性能、高可用的服务器一样。客户程序不受服务器集群的影响不需作任何修改。系统的伸缩性通过在服务机群中透明地加入和删除一个节点来达到，通过检 测节点或服务进程故障和正确地重置系统达到高可用性。由于我们的负载调度技术是在Linux内核中实现的，我们称之为Linux虚拟服务器（Linux Virtual Server）。

* Lvs最常用类型-DR模型（直接路由）
![VS/NAT的体系结构][1]


----------


**步骤**

* 模型构建：node3为虚拟IP服务器，node1和node2为转发服务器；所以node3配置VIP，node1和node2配置网卡接口

>
* 创建虚拟IP
>ifconfig eth0:1 192.168.116.200/24

* a)两台分发服务器：静态设置
* 修改网络配置参数
	* arp_ignore: 定义接收到ARP请求时的响应级别：
		* 0：只要本地配置的有相应地址，就给予响应；
		* 1：仅在请求的目标(MAC)地址配置请求到达的接口上的时候，才给予响应；
	* arp_announce：定义将自己地址向外通告时的通告级别；
		* 0：将本地任何接口上的任何地址向外通告；
		* 1：试图仅向目标网络通告与其网络匹配的地址；
		* 2：仅向与本地接口上(MAC)地址匹配的网络进行通告；

* b)修改报文源IP的设置，需要设置内核参数

``` 查看并修改arp_ignore内容为1，arp_announce为2
more /proc/sys/net/ipv4/conf/eth0/arp_ignore
echo "1" >/proc/sys/net/ipv4/conf/eth0/arp_ignore
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
echo "2" >/proc/sys/net/ipv4/conf/eth0/arp_announce
```

* c)在两台机器（RS）上，设置网卡的别名IP：192.168.116.200
>ifconfig lo:0 192.168.116.200 netmask 255.255.255.255 broadcast 192.168.116.200
![设置网卡别名][2]

* d)在两台机器（RS）上，添加一个路由
>route add -host 192.168.116.100 dev lo:0

* 1、	找一台主机作为DR（虚拟服务器），安装ipvsadm、httpd
>Yum install ipvsadm

* RS配置hosts
> vi /etc/hosts
![添加地址][3]

* 测试：访问RS：192.168.116.3,出现下面页面
![页面][4]

* 命令：网络拷贝
>scp /etc/hosts root@node2:/etc/

* 默认访问的是官方页面，在`/var/www/html/`下天机index.html自己的文件，将访问此文件
![index.html][5]
![node2（node1同样）][6]

* 之后访问192.168.116.3或者192.168.116.4;分别返回自己的页面
![node1][7]
![node2][8]

**整个原理**
![网络配置][9]

**Ipvsadm命令：**
*管理集群服务

``` stylus
添加：-A -t|u|f service-address [-s scheduler]
			-t: TCP协议的集群 
			-u: UDP协议的集群
				service-address:     IP:PORT
			-f: FWM: 防火墙标记 
				service-address: Mark Number
		修改：-E
		删除：-D -t|u|f service-address

		# ipvsadm -A -t 172.16.100.1:80 -s rr

```
![添加集群服务][10]
* 管理集群中的RS

``` stylus
添加：-a -t|u|f service-address -r server-address [-g|i|m] [-w weight]
			  -t|u|f service-address：事先定义好的某集群服务
			  -r server-address: 某RS的地址，在NAT模型中，可使用IP：PORT实现端口映射；
			  [-g|i|m]: LVS类型	
				-g: DR
				-i: TUN
				-m: NAT
			[-w weight]: 定义服务器权重100
		修改：-e
		删除：-d -t|u|f service-address -r server-address

```


![添加RS][11]

* 最终效果：访问VIP：192.168.116.200,
![node1][12]
![node2][13]
![验证][14]

**注意**这里关键的核心就两步：1、虚拟服务器ipvsadm配置转发；2、多个RS配置网卡和参数
前一天出现一些错误，导致无法访问的问题，之后参考下下面的连接重新做了下，成功了，如上图。[肖邦linux的博客][15]
* 问题排查步骤参考：[gmoon23的博客][16]（时时注意防火墙问题，几次了）


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521469636829.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521532696322.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521535527208.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521535592731.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521536208507.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521536227494.jpg
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521536277540.jpg
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521536288454.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521536499274.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521548406143.jpg
  [11]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521548528375.jpg
  [12]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521599530976.jpg
  [13]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521599556872.jpg
  [14]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521600591104.jpg
  [15]: https://www.cnblogs.com/liwei0526vip/p/6370103.html
  [16]: http://blog.csdn.net/gmoon23/article/details/75379863
