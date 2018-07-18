---
title: linux搭建java学习几个问题处理
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

 1. linux搭建java开发环境
 2. linux上运行.jar文件


----------
* 1、linux搭建java开发环境
	* 1.1、官网下载tar包
	* 1.2、创建java文件夹：`cd /usr --> mkdir java`
	* 1.3、通过`mv jdk1.8.tar /usr/java`，将jdk.tar移动到java中；`tar -zxvf jdk-8u60-linux-x64.tar.gz`移动到当前文件夹
	* 1.4、配置环境变量：`gedit ~/.bashrc`在打开的文件的末尾添加`
export JAVA_HOME=/usr/lib/jvm/jdk7
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH`
保存退出，然后输入下面的命令来使之生效`source ~/.bashrc`
	* 1.5、配置默认jdk：由于一些Linux的发行版中已经存在默认的JDK，如OpenJDK等。所以为了使得我们刚才安装好的JDK版本能成为默认的JDK版本，我们还要进行下面的配置。执行下面的命令：`
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk7/bin/java 300
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk7/bin/javac 300`
注意：如果以上两个命令出现找不到路径问题，只要重启一下计算机在重复上面两行代码就OK了。执行下面的代码可以看到当前各种JDK版本和配置：
`sudo update-alternatives --config java`
	* 1.5、测试：`java -version`


----------


* 2、linux上运行.jar文件
	* 2.1、通过eclipse的export导出.jar文件
	* 2.2、一种是`java –classpath test.jar packageNmae.HelloWorld`,另一种修改导出的jar包中的清单文件。具体地，通过压缩工具打开test.jar,找到MANIFEST.MF文件，打开并添加Main-Class: packageName.HelloWorld，注意冒号后面有一个空格，而且这里添加的是HelloWorld的完整包路径，最后在HelloWorld后面回车，让光标定位到下一行，一定要回车！


----------


参考链接：[linux下配置jdk][1]；[linux下运行jar][2]


  [1]: http://www.linuxidc.com/Linux/2013-11/93012.htm
  [2]: https://wenku.baidu.com/view/c5cf1a0d02768e9950e73845.html
