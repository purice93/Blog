---
title: linux下非root用户安装软件-pyaudio、sounddevice
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>最近在配置一个深度学习框架，由于服务器是公用的，导致并没有root权限，所有对于许多的软件安装，都无法直接使用apt-get安装，因此需要采取编译安装的方式，暂且先记下来，以后作为参照。主要的负载点并不在于安装步骤，而在于如何处理其中的依赖关系。由于安装时间已经过了一段时间，一直无闲暇顾忌，所以可能存在有些步骤遗漏。

安装之前首先推荐两个python库下载地址：
win+linux版：https://pypi.org/simple
只有windows版：https://www.lfd.uci.edu/~gohlke/pythonlibs/

主要是用第一个地址，第二个地址备用吧

1. 下载安装库：[portaudio19_19.6.0.orig.tar.gz][1]
2. 解压：`tar - zxvf Pportaudio19_19.6.0.orig.tar.gz`
3. **环境配置：**
	切换到解压后的目录，运行 ./configure。./configure –help可以列出配置项，非root用户最重要的配置项是安装目录prefix，例如 ./configure –prefix=/home/yourname/bin。在无法自动找到依赖库位置的情况下，用 –with-xx-dir=xxx 的形式配置依赖库位置；
4. 编译并安装：`make &&  make install`
5. 添加PATH变量到自己的用户文件
	``` stylus
	vim ~/.bashrc
	export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/zoutai/bin/lib 
	export LIBRARY_PATH=/home/zoutai/bin/inctude/:$LIBRARY_PATH 
	CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/home/zoutai/bin/inctude:/MyLib 
	export CPLUS_INCLUDE_PATH
	```

python库的编译安装：
1. 解压pyaudio、sounddevice
2. 构建：`python setup.py build`
3. 安装：`python setup.py install`

上述过程有部分可能报错，可以更具报错条件，网上查找，由于细节我已忘记，所以没在写，下次记起来再补充。





  [1]: http://portaudio.com/download.html
