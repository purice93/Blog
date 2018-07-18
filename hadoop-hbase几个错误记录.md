---
title: hadoop-hbase几个错误记录
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>集群这东西，就是各种文件配置，太过于杂乱导致，如果你不是特别细心，总是会错误百出，以致于精神萎靡而无法向前。但是，有时候，即便你细心得像个暖男，最后还是会出现问题，很多时候，这并不是你的错，而是机器的错，但是，如果你不能够在短时间内找到“问题”的真正原因，背锅的还是你自己

**几个浪费时间的问题**
* 1还是防火墙问题：把防火墙全部给永久封停
```
# 关闭iptables
/etc/init.d/iptables stop

# 永久关闭
chkconfig iptables off

# 检查
chkconfig --list iptables
```
* 2免密登录问题：参考博客[https://blog.csdn.net/syani/article/details/52618840][1]
* 3stopping hbasecat: /tmp/hbase-root-master.pid: No such file or director（hbase无法停止）
在hbase-env.sh中修改pid文件的存放路径，默认路径容易丢失，配置如下：[参考博客][2]
```
# The directory where pid files are stored. /tmp by default.

export HBASE_PID_DIR=/var/hadoop/pids 
```
* 4hbase集群部分节点HRegionServer启动后自动关闭的问题
集群节点的时间不一致！服，又是这个问题，之前改过！解决方法如下：
```
1）在hbase-site.xml文件中 修改增加 ，将时间改大点（我采用的，懒得改时间了）
<property>
<name>hbase.master.maxclockskew</name>
<value>150000</value>
</property>
2）或者修改系统时间，将时间改为一致（建议采用本方法）：（这种方式不精确，需要使用一个远程节点作为校准比较好）
修改日期
date -s 11/23/2013
修改时间
date -s 15:14:00
检查硬件（CMOS）时间
clock -r
将系统时间写入CMOS
clock -w

3、修改完成后单独启动HRegionServer节点即可：
启动集群中所有的regionserver
./hbase-daemons.sh start regionserver
启动某个regionserver
./hbase-daemon.sh start regionserver
```
* 5 主节点node1可能中途死掉重启，从active变为standby，导致hbase死掉；需要关闭重启node2辅助节点，让node1重新变为active**时时刻刻都要检查每个点是否出错**



* hbase使用
使用phonenix批量导入csv数据到hbase数据库
```
bin/psql.py node1:2181 建表sql 数据文件csv
```
![enter description here][3]


  [1]: https://blog.csdn.net/syani/article/details/52618840
  [2]: https://blog.csdn.net/qq_20545159/article/details/49454335
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1526095617877.jpg
