---
title: 第六章-session一致性的另外一种方案tengine的会话保持功能：使用浏览器cookie来保存会话对应的服务器
tags: cookie,session,tengine
grammar_cjkRuby: true
---

>分布式服务器是为了解决高并发而存在，但是却会出现session一致性的问题，之前的方案主要是通过缓存数据库来共享session，而这种方式的效率却并不高，因为存在一个中间过程问题。但是在某些情况下，由于服务器宕机的可能性很低，或者说并不需要高并发的情况下，并不需要通过多余的缓存数据服务器来共享session。比如，一个用户通过浏览器登录淘宝网，其实对于一个用户而言，一个浏览器的访问数量很少，完全不必要使用多个服务器来服务。所以，对于单个指定用户，可以直接记录这个用户的第一个服务器id，之后nginx总是将此用户的请求发个同一个服务器，而不需要另外的redis缓存数据服务器。

* nginx配置

``` stylus
会话保持模块配置
–#insert + indirect模式：
–upstream test {
–session_stickycookie=uiddomain=www.xxx.comfallback=on path=/ mode=insert option=indirect;
–server 127.0.0.1:8080;
–}
–Server
–{
–location / {
–#在insert + indirect模式或者prefix模式下需要配置session_sticky_hide_cookie
–#这种模式不会将保持会话使用的cookie传给后端服务，让保持会话的cookie对后端透明
–session_sticky_hide_cookieupstream=test;
–proxy_passhttp://test;
–}
–}
```
![配置图][1]

* 重启nginx：`service nginx reload`
* 效果：页面将不发生变化
![第一次访问][2]
![刷新后不变化][3]

* vim跳转指定行
	* :$ 跳转到最后一行
	* :1 跳转到第一行

* xshell卡死时：Ctrl+Q


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521448808403.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521450486644.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521450557452.jpg
