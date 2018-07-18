---
title: 实验楼linux学习（1-2节） 
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
1. 学习路径
	* 初级：系统安装、图形界面使用、常用命令
	* 中级：用户和文件系统管理、软件安装和配置、网络管理、系统维护、shell编程初步
	* 高级：
		* 服务器：数据库、防火墙、dns-vpn-web-mail-ftp-lamp-nfs-samba服务器
		* 程序开发：shell高级编程、c/c++开发、内核基础、嵌入式开发、linux驱动开发
		* 


----------


2. 基本概念和操作
	* 第一节、几个重要命令：
		`tap键	-- 智能提示和补全`
		`ctrl+c	-- 强行终止:Ctrl+d	键盘输入结束或退出终端`
		`↑上下箭头用来查看历史命令`
		`man	查看命令详细信息和例子`
		
	* 第二节、用户和文件权限：
		* su -l <user> --切换用户
		* sudo adduser <username> --创建用户，同时为新用户创建home目录
		* groups <username> --查看用户所在的用户组
		* cat /etc/group | grep -E "shiyanlou" --前部分查看所有的用户组，后半部分筛选用户组，| sort为按字典排序
		* sudo usermod -G sudo lilei --通过su lilei 切换到当前用户，然后通过此命令将lilei添加到sudo用户组，第一个sudo是权限命令
		* sudo deluser lilei --remove-home --删除用户及其空间
		* ls 查看所在目录所有文件，exit --退出当前用户	
		* $ cd /home/lilei-$ ls iphone6-$ sudo chown shiyanlou iphone6 --更改文件iphone6所有者为shiyanlou
		* 修改文件权限，文件权限为9位二进制数，有三个三位，每个分别代表当前用户-所属用户组-其他用户，三位分别代表rwx，即读写改，最终通过十位数来表示，如下图：![二进制表示文件权限][1]
		* chmod 700 iphone6 -- 修改iphone文件权限
		* gou-rwx iphone6 -- 参数gou分别表示group-other-user,+-表示添加删除权限
		* adduser 和 useradd 的区别是什么？useradd只创建用户，不管其他；adduser创建用户的同时，创建密码和创建空间home
		* touch也可以创建文件，vim也行，gedit可以编辑文件-同记事本

3. 练习2
	* ssh远程登录服务器
		* 登录服务器:`ssh 用户名@IP地址 -p 端口号`
		* 上传文件夹：`scp -P 端口号 本地文件路径 用户名@远程服务器地址:远程路径`(需要在本地用户操作)



  [1]: ./images/1505870132818.jpg
