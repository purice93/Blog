---
title: nginx与tomcat进行服务器方向代理-负载均衡
tags: nginx,tomcat,反向代理,负载均衡
grammar_cjkRuby: true
---

**反向代理图：**
![服务器方向代理][1]


**1、转发**
>即通过nginx将请求转发给tomcat服务器
* 配置nginx*（使用的是阿里的tengine）服务器为192.168.116.3:80（系统默认为80端口，所以直接访问192.168.116.2即可）
![nginx访问页面][2]
* 配置tomcat服务器为192.168.116.3:8080
![tomcat访问页面][3]

* 注意：每次重启虚拟机都需要重新关闭虚拟机防火墙，否则无法从主机访问虚拟机服务器
>service iptables stop

* 修改nginx配置
>vi /opt/sxt/soft/tengine-2.1.0/conf/nginx.conf
>在server的location出添加proxy_pass属性，指定到转发服务器
![修改nginx配置][4]

* 重新访问nginx服务器，将转发到tomcat服务器
![nginx转发到tomcat][5]


----------


**2、负载均衡**
>对并发请求进行分发，使得每一个tomcat服务器接受到的请求近似相同，达到均衡

* 修改配置：
>默认每个tomcat权重为1，即进行轮换访问
![负载均衡][6]
![第一次访问为tomcat1][7]
![刷新后变为tomcat2][8]


----------
**健康检查**
>添加健康检查配置，可以查看服务器的状况，这样，便于服务器维护和记录查询
>注意，直接从word格式粘贴会报错，注意空格和换行符
* 添加检查配置
![添加两项配置][9]
* 通过http://192.168.116.3/status就可以访问检查面板
![检查面板界面][10]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521280258545.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521278599708.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521279852315.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521280095492.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521280190955.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521284140283.jpg
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521284457409.jpg
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521284475889.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521287322436.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521287524551.jpg
