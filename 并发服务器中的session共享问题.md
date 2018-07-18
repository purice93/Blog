---
title: 分布式服务器中的session共享问题
tags: 分布式,session共享,memcached
grammar_cjkRuby: true
---

>为了使得分发后的多个tomcat服务器，可以对请求session进行共享，我们需要使用一个特殊的数据服务器。
>一般有memcached和redis3。（redis没有这个功能）

**session共享原理**
* http协议是无状态的，即你连续访问某个网页100次和访问1次对服务器来说是没有区别对待的，因为它记不住你。那么，在一些场合，确实需要服务器记住当前用户怎么办？比如用户登录邮箱后，接下来要收邮件、写邮件，总不能每次操作都让用户输入用户名和密码吧，为了解决这个问题，session的方案就被提了出来，事实上它并不是什么新技术，而且也不能脱离http协议以及任何现有的web技术
* session的常见实现形式是会话cookie（session cookie），即未设置过期时间的cookie，这个cookie的默认生命周期为浏览器会话期间，只要关闭浏览器窗口，cookie就消失了。实现机制是当用户发起一个请求的时候，服务器会检查该请求中是否包含sessionid，如果未包含，则系统会创造一个名为JSESSIONID的输出cookie返回给浏览器(只放入内存，并不存在硬盘中)，并将其以HashTable的形式写到服务器的内存里面；当已经包含sessionid是，服务端会检查找到与该session相匹配的信息，如果存在则直接使用该sessionid，若不存在则重新生成新的session。这里需要注意的是session始终是有服务端创建的，并非浏览器自己生成的。但是浏览器的cookie被禁止后session就需要用get方法的URL重写的机制或使用POST方法提交隐藏表单的形式来实现
* Session共享：
首先我们应该明白，为什么要实现共享，如果你的网站是存放在一个机器上，那么是不存在这个问题的，因为会话数据就在这台机器，但是如果你使用了负载均衡把请求分发到不同的机器呢？这个时候会话id在客户端是没有问题的，但是如果用户的两次请求到了两台不同的机器，而它的session数据可能存在其中一台机器，这个时候就会出现取不到session数据的情况，于是session的共享就成了一个问题
* Session一致性解决方案：
	* 1、session复制
	 tomcat 本身带有复制session的功能。（不讲）
	* 2、共享session
	需要专门管理session的软件，
	memcached缓存服务，可以和tomcat整合，帮助tomcat共享管理session。


----------
**利用memcached实现session共享**

* 安装mencached
>yum -y install memcached
>service memcached start #启动memcached服务器
* 复制memcached相关的jar包到tomcat的lib下[下载连接][1]
![memcached相关jar包][2]
* 修改tomcat标记`vi conf/server.xml`
![添加tomcat标记，用于访问标志][3]
* vim中的查找命令`/tag`;即命令行模式键入"/","tag"即为所需查找的字符串
* 修改tomcat默认访问页面index.jsp，加入session
![修改默认界面index.jsp，添加session支持][4] 
* 未添加一致行前，访问nginx时，将产生多个session
![tomcat1][5]
![tomcat2][6]
* 添加一致性，tomcat连接memcached，将session存入memcached
>vi conf/context.xml
``` stylus
<Manager className="de.javakaffee.web.msm.MemcachedBackupSessionManager" 
	memcachedNodes="n1:192.168.116.3:11211" 
    sticky="false" 
    lockingMode="auto"
    sessionBackupAsync="false"
	requestUriIgnorePattern=".*\.(ico|png|gif|jpg|css|js)$"
    sessionBackupTimeout="1000" transcoderFactoryClass="de.javakaffee.web.msm.serializer.kryo.KryoTranscoderFactory" 
/>
```
![添加memcached连接][7]
* 分别修改tomcat对应的index.jsp

``` stylus
<%@ page language="java" contentType="text/html; charset=UTF-8"  pageEncoding="UTF-8"%>
SessionID:<%=session.getId()%>
<br/>
SessionIP:<%=request.getServerName()%> 
<br/>
<h1>tomcat2 page</h1>
```
* 重启memcached、nginx和tomcat，访问`192.168.116.3`
* 效果：sessionID不变，只是服务器页面变化
![tomcat1][8]
![tomcat2][9]

**注意事项：过程中可能存在各种各样的问题，导致错误。如复制空格问题。当实在无法找到错误原因时，可以文件整体还原，在修改，因为有时候可能是自己错误修改了文件，但是找不到地址了**

----------


完整参考流程：

``` stylus
1.	安装memcached内存数据库
yum –y install memcached
可以用telnet localhost 11211
Set abc 0 0 5
12345
get abc 
2.	web服务器连接memcached的jar包拷贝到tomcat的lib
3.	修改server.xml里面修改Engine标签，添加jvmRoute属性，目的是查看sessionid里面带有tomcat的名字，就是这里配置的jvmRoute
<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat1">
4.	配置tomcat的conf目录下的context.xml
<Manager className="de.javakaffee.web.msm.MemcachedBackupSessionManager"
        memcachedNodes="n1:192.168.17.9:11211"
    sticky="false"
    lockingMode="auto"
    sessionBackupAsync="false"
        requestUriIgnorePattern=".*\.(ico|png|gif|jpg|css|js)$"
sessionBackupTimeout="1000" transcoderFactoryClass="de.javakaffee.web.msm.serializer.kryo.KryoTranscoderFactory" />
配置memcachedNodes属性，配置memcached数据库的ip和端口，默认11211，多个的话用空格隔开
目的？让tomcat服务器从memcached缓存里面拿session或者是放session
5.	修改index.jsp，取sessionid看一看
<%@ page language="java" contentType="text/html; charset=UTF-8"  pageEncoding="UTF-8"%>
<html lang="en">
SessionID:<%=session.getId()%>
</br>
SessionIP:<%=request.getServerName()%>
</br>
<h1>tomcat1</h1>
</html>

```


**利用redis实现session共享**

* 安装redis `yum -y install redis`
可能会存在无yum源的问题：

``` stylus
CentOS默认的安装源在官方的centos.org上，而redis在第三方的yum源里，因此无法安装。这就是我们常常在yum源里找不到各种软件的原因，还需要自己去wget，然后configure,make,make install，这个过程太痛苦了，并且卸载软件的时候还容易出错。
非官方的yum推荐用fedora的epel仓库。epel (Extra Packages for Enterprise Linux)是基于Fedora的一个项目，该仓库下有非常多的软件，建议安装。yum添加epel源的命令为：yum install epel-release然后回车。安装完后，我们使用命令：yum repolist查看。然后重新安装redis
```
* 导入相关的redis-tomcat的jar包到tomcat中（下载链接同上）
![三个依赖jar包][10]
* 检测是否安装正确
>由于redis默认的打开端口是6379；memcached默认端口是11211；所以可以直接通过telnet来检查端口情况：`telnet localhost 6379`

* 修改redis相关配置`vi /etc/redis.conf`，将bind的127.0.0.1修改为本机地址，否则只能本机访问了。会出现各种错误！
>另外注意：修改的bind为第二个不是第一个！
![不是这里的][11]
![而是这里][12]

* 添加redis配置到tomcat中，同memcached一样

``` stylus
<Valve className="com.orangefunction.tomcat.redissessions.RedisSessionHandlerValve" />
<Manager className="com.orangefunction.tomcat.redissessions.RedisSessionManager"
         host="192.168.116.3"
         port="6379"
         database="0"
         maxInactiveInterval="60" />
```
* 测试效果
![tomcat1][13]
![tomcat2][14]


  [1]: https://code.google.com/archive/p/memcached-session-manager/downloads
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521338268310.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521338516695.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521340498851.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521349650737.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521342012606.jpg
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521349290610.jpg
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521349570726.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521349582665.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521382915270.jpg
  [11]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521446257269.jpg
  [12]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521446313973.jpg
  [13]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521446558722.jpg
  [14]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521446569528.jpg
