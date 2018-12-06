---
title: 非root用户安装linux软件
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

>最近经常用到服务器，但是实验室服务器都是公用的，自己并没有root权限。导致很多时候，需要某些软件，不能直接使用apt-get命令安装。

案例1：tree命令的安装

1. 直接下载tar压缩包、解压到指定目录：
>wget http://mama.indstate.edu/users/ice/tree/src/tree-1.7.0.tgz
>tar -zxvf ~

2. ls查看安装文件

![查看文件目录及说明](http://osiy4s0ad.bkt.clouddn.com/soundblog/1541143435527.png)

3. 修改配置文件安装地址
默认地址为root用户地址，安装无权限

>vi Makefile  # 修改配置文件

![修改配置文件](http://osiy4s0ad.bkt.clouddn.com/soundblog/1541143628799.png)

4. 编译安装
>make 
>make MANDIR=/home/zoutai/share/man/man1 install && chmod -v 644 /home/zoutai/share/man/man1/tree.1

说明：只使用make，后续会报错，需要加上第二句

> make install # 安装

另外：如果安装出错，需要使用`make clean`清除之前的错误编译

5. 将安装软件加入到环境变量，并激活
>vi ~/.bashrc
加入：`export PATH="/home/zoutai/bin/tree:$path"`

>source ~/.bashrc

相关参考：
https://www.jianshu.com/p/f7f24b4b2625
