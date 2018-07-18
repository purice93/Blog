---
title: linux的安装与克隆
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

克隆：
* 物理克隆
	* 鼠标右键-管理-克隆
	* 选择完整克隆
	![enter description here][1]
	* 设置安装目录
	![enter description here][2]

* 修改配置
	* 克隆之后的操作系统需要重新分配物理地址
		* 删除/etc/sysconfig/network-scripts/ifcfg-eth0 文件中的物理地址：`删除两行：UUID和物理地址`
		* 删除文件/etc/udev/rules.d/70-persistent-net.rules`rm -rf /etc/udev/rules.d/70-persistent-net.rules`
		上述/etc/sysconfig/network-scripts/ifcfg-eth0文件可能有所不同，比如我的并没有上面两行，但是有个IPADDR，修改这个
		![enter description here][3]
		
* xshell新建连接
	* 配置主机名和ip
	![enter description here][4]
	* 配置登录名和密码
	![enter description here][5]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526304858753.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526304921642.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526305178596.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526305275834.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526305335807.jpg
