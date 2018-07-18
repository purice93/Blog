---
title: MongoDB学习记录
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

数据库和集合基本操作
一、实验介绍
1.1 实验内容
本次实验将介绍 MongoDB 的基本概念，以及数据库和集合的基本操作。

1.2 实验知识点
MongoDB 概念
数据库基本操作
集合基本操作
1.3 实验环境
课程使用的实验环境为 Ubuntu Linux 14.04 64 位版本。实验中会用到程序：

MongoDB 2.4.9
Xfce终端
二、实验步骤
本教程只介绍了 MongoDB 的基础知识，其相关的安装配置等并未涉及，如有需要可参考 MongoDB环境配置教程。

2.1 MongoDB 简介
2.1.1 简介
MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。它支持的数据结构非常松散，是类似 json 的 bson 格式，因此可以存储比较复杂的数据类型。

2.1.2 面向集合的存储
在 MongoDB 中，一个数据库包含多个集合，类似于 MySQL 中一个数据库包含多个表；一个集合包含多个文档，类似于 MySQL 中一个表包含多条数据。

2.1.3 虚拟机开机配置
启动 MongoDB 服务，因为 MongoDB 并不随系统一起启动，可能以下命令运行后会等一小段的时间才会启动完毕。

$ sudo service mongodb start
进入 MongoDB 命令行操作界面(可能会出现 connect failed，多试几次就行)，在命令行中敲exit可以退出。

$ mongo
实验中的布尔类型的 ture 用1代替，false 用0代替。

2.2 基本概念
2.2.1 数据库
一个 MongoDB 可以创建多个数据库
使用 show dbs 可以查看所有数据库的列表
执行 db 命令则可以查看当前数据库对象或者集合
运行 use 命令可以连接到指定的数据库
$ mongo      #进入到mongo命令行
> use test            #连接到test数据库
注意：数据库名可以是任何字符，但是不能有空格、点号和$字符

2.2.2 文档
文档是 MongoDB 的核心，类似于 SQLite 数据库（关系数据库）中的每一行数据。多个键及其关联的值放在一起就是文档。在 Mongodb 中使用一种类 json 的 bson 存储数据，bson 数据可以理解为在 json 的基础上添加了一些 json 中没有的数据类型。

例：

{"company":"Chenshi keji"}
2.2.3 文档的逻辑联系
假设有两个文档：

{
   "name": "Tom Hanks",
   "contact": "987654321",
   "dob": "01-01-1991"
}#user文档

{
   "building": "22 A, Indiana Apt",
   "pincode": 123456,
   "city": "chengdu",
   "state": "sichuan"
}#address文档
关系1：嵌入式关系，把 address 文档嵌入到 user 文档中

{
   "name": "Tom Hanks",
   "contact": "987654321",
   "dob": "01-01-1991",
   "address":
   [{
   "building": "22 A, Indiana Apt",
   "pincode": 123456,
   "city": "chengdu",
   "state": "sichuan"
    },
    {
    "building": "170 A, Acropolis Apt",
    "pincode": 456789,
    "city": "beijing",
    "state": "beijing"
    }]
}#这就是嵌入式的关系
关系2：引用式关系：将两个文档分开，通过引用文档的_id字段来建立关系

{
   "contact": "987654321",
   "dob": "01-01-1991",
   "name": "Tom Benzamin",
   "address_ids": [
      ObjectId("52ffc4a5d85242602e000000")    #对应address文档的id字段
   ]
}#这就是引用式关系
2.2.4 集合
集合就是一组文档的组合，就相当于是关系数据库中的表，在 MongoDB 中可以存储不同的文档结构的文档。

例:

{"company":"Chenshi keji"} {"people":"man","name":"peter"}
上面两个文档就可以存储在同一个集合中。

2.2.5 元数据
数据库的信息存储在集合中，他们统一使用系统的命名空间：DBNAME.system.*

DBNAME 可用 db 或数据库名替代

DBNAME.system.namespaces ：列出所有名字空间
DBNAME.system.indexs ：列出所有索引
DBNAME.system.profile ：列出数据库概要信息
DBNAME.system.users ：列出访问数据库的用户
DBNAME.system.sources ：列出服务器信息
2.3 数据库的创建和销毁
2.3.1 创建数据库
启动服务后，进入 MongoDB 命令行操作界面：

$ mongo
使用 use 命令创建数据库：

> use mydb
查看当前连接的数据库：

> db
查看所有的数据库：

> show dbs
列出的所有数据库中看不到 mydb 或者显示 mydb(empty) ，因为 mydb 为空，里面没有任何东西，MongoDB 不显示或显示 mydb(empty)。

2.3.2 销毁数据库
使用 db.dropDatabase() 销毁数据库：

> use local
 switched to db local
> db.dropDatabase()
查看所有的数据库：

> show dbs
2.4 集合（collection）的创建和删除
2.4.1 创建集合
在数据库 mydb 中创建一个集合

> use mydb
switched to db mydb
> db.createCollection("users")
查看创建的集合：

> show collections
2.4.2 删除集合
删除集合的方法如下：（删除 users 集合）

> show collections
> db.users.drop()
查看是否删除成功：

> show collections
2.5 向集合中插入数据
2.5.1 使用 insert()
插入数据时，如果 users 集合没有创建会自动创建。

> use mydb
switched to db mydb
> db.users.insert([
... { name : "jam",
... email : "jam@qq.com"
... },
... { name : "tom",
... email : "tom@qq.com"
... }
... ])
2.5.2 使用 save()
插入数据时，如果 users 集合没有创建会自动创建。

> use mydb2
switched to db mydb2
> db.users.save([
... { name : "jam",
... email : "jam@qq.com"
... },
... { name : "tom",
... email : "tom@qq.com"
... }
... ])
三、实验总结
本节实验介绍了 MongoDB 和集合的基本操作，在 MongoDB 中使用一种类 json 的 bson 存储数据，可以使用 use 创建和切换数据库，show dbs 可以查看有哪些数据库，dropDatabase 可以删除数据库，createCollection 可以创建集合，show collections 可以查看集合，insert() 和 save() 可以插入数据。

四、课后习题
请新建一个名为 shiyanlou 的数据库，创建一个 users 的集合，插入一个name：你的昵称的文档，并了解 insert 和 save 的区别。

五、参考链接
本实验课程参考以下文档：

MongoDB 官方教程

来源: 实验楼
链接: https://www.shiyanlou.com/courses/12
本课程内容，由作者授权实验楼发布，未经允许，禁止转载、下载及非法传播
