---
title: mysql必知必会-基础再复习笔记1
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>在大部分场景下，基本的curd操作就已经满足要求。而且对于不是专门从事数据库开发的人而言，数据库其实应该遵从“用时即查”的方式。并不需要太过于系统的学习。但是，接触的实际操作太少，使得我门并不能很好地练习数据库。另外，数据库并不是单单的数据库，这其中的技术不仅仅设计复杂的数据结构的应用，还涉及到各种工程化的技术，多线程优化，等等。基本上，数据库的很多构建逻辑，已经包含了很多实际用到的其他技术。


----------
>为了进一步加深对数据库的知识体系的认识,暂且做一个mysql的总体笔记,包含所有的知识构建,参考书以《mysql必知必会》。


### 数据库学习路线
1. 数据库基础概念：《数据库系统概念》Abraham Silberschatz  作为数据库入门的基础读物，本书可以快速的帮助计算机专业学生了解数据库，连接数据库的用处，基本构建原理，基本分类及其作用场景，基本操作等等，他是所有数据库如mysql的基本原型，其他的数据库都是根据其理论生产的现成软件。
2. 数据库基本操作（以下都以mysql为例）：《MYSQl必知必会》，我最初用的是《MYSQL从入门到精通》，这个阶段书籍其实都差不多，你只需要学会简单的增删改查就可以，并不需要了解太多的复杂结构，基本上，学到这里，就可以进行实战了。
3. 数据库进阶：《sql进阶教程》，当我们熟练使用基本的操作语句后，我们开始不满足了。原因在于，随着我们的软件上线，遇到了实际环境，数据量增大，很多效率问题都会暴露出来；另外，随着项目功能的不断添加，许多复杂的功能，仅仅使用简单地数据操作已经完成不了，所以，这时候需要使用复杂的操作，比如各种聚合。
4. 数据库原理：《高性能MYSQl》+《MySQL技术内幕 InnoDB存储引擎-第2版》。之所以把底层理论部分放在最后，是因为，我个人觉得，很多时候，我们在学习**使用**一个技术的时候，第一步并不是去弄懂它的底层，虽然大部分高校的教学模式确实是这样，真正这样做的也确实有很多好处，但是，对于很多人而言，这样的摸不着头脑的过程很容易让初学者望而却步，我只是个开车的，为什么非得去学习造车，这让我觉得开车好难啊。所以，第一步，先学会用，用熟了再慢慢去摸索原理，就很容易了。就像大部分老司机现在基本上都能修一修自己的车。
数据库原理这块，其实很重要，这里面完美的应用了很多数据结构的知识，其实这里就是数据结构最好的应用。所以，网上有时候在讨论为什么面试对数据结构看得这么重要，开发个网站并不需要这丫？其实这是因为你开发的网站太菜了，遇到的问题也太菜了，所以还没有到达那个地步。但是，工程师却要未雨绸缪，指不定哪天你就要电商网站也要遇到双十一问题。


----------


1. sql基本概念
数据库database：即存储数据的仓库；
表table：仓库里面的每一个箱子，箱子的样式有很多种
模式schema：
列column：箱子的抽屉的格式，字段类型，如id、name
行row：一行数据
基本数据类型datatype：
主键primary key:必须定义主键；主键唯一；不能为null；不能重复；不能更改变化（比如商品价格）。
sql：sql即structured query language，结构化查询语言

2. 基本操作
2.1. 数据库查询：
显示数据库：show database;
进入数据表：use tablename;
2.2. 查询数据：
select id from tablename；
select * from tablename；尽量不要使用全查找*：要求不明确，效率低。
限制条件：
不重复：distinct (类似于Python中的set函数)
分页：limit n,m或者limit m offset n：表示从偏移量第n个取（不包括n），取m条数据
2.3 数据排序
order by salary DESC；
指定排序方向：默认从小到大即正序：ASC；倒序需指定为DESC
2.4 过滤数据/条件判断
where id=10086
数据过滤也可以通过客户端判断，但这样将大大增加时间。
另外where和order by组合使用时，需要将where放在前面，否则出错
几个常用条件：不等于<>或!=；之间between and或者in(100,200)；空值检查is null;
组合条件：and、 or、 not
2.5 通配符匹配过滤wildcard
like操作符：
%：匹配任何字符任何长度
_：同%一样，但是下划线只能匹配单个字符
例子：`select name from tablename where name like 'zhang%'`
[]：匹配括号中的任意一个字符；[JM]%：即匹配J或M开头的数据
^：否定。[^JM]%：不匹配J或M开头的数据
**通配符注意事项**
* 尽量不要使用通配符，因为通配符的查找速度太慢，基本是遍历查找，比较的是每个数据字符，而不是整个数据
* 通配符尽量使用在后面部分，如果放在开头一样大大降低搜索速度，这个原理和多字段索引类似
2.6 正则表达式
关键词regexp：`select name from tablename where name REGEXP 'zhang'`字段将匹配名字中含有zhang的数据，这里与like全局匹配不同
另外正则中.表示匹配任意一个字符；另一个不同是like不区分大小写，但是regexp区分大小写
or：使用|：regexp '1000|2000'
匹配几个字符之一：regexp '[123]Ton'
匹配范围：regexp '[01234]'或[0-4]；[a-z]
匹配特殊字符：._等字符有特殊汉子，需要转义；使用'\\.'对点号进行转义
匹配多个字符：
![enter description here][1]
定位符：
![enter description here][2]
2.7 数据聚合/字段计算
字段拼接concatenate：concat(firstname,lastname)
删除字段左右多余的空格：RTrim(name)，LTrim(name)
使用别名：select firstname AS name
算数计算：支持+-*/计算
2.8 使用数据处理函数
* 文本处理函数
	* 大小写转换：Upper/Lower
	* 长度：Length
* 日期处理
	* 获取日期年月日：Year、Month、Day
* 数值处理：
	* 绝对值Abs、除数取余Mod等
2.9 数据处理
* 数据汇总
	* 平均值AVG、总数COUNT、最大值最小值MAX/MIN、总和SUM
* 数据分组：group by
分组过滤：having；having基本上能够替代所有的where
分组和排序：group和order：分组大部分和聚合函数一起使用，只是为了显示固定的数据，order是主要为了排序；所以有时候不要依赖group进行排序，还需要order语句
* select语句顺序
select from where groupby having orderby limit
2.10 使用子查询
select查询出来的数据可以作为临时表，这样可以通过临时表来进行数据过滤：`select name from tablename1 where id in (select id from tablename2)`
2.11 联结表join
内连接：多个表进行联结有两种方式，一种直接使用select * from table1，table2;另一种使用inner join on：select * from table1 inner join table2 on table1.id = table2.id
2.12 高级联结
外链接：左右连接left join 和right join；之间的区别就是，左连接将显示所有的左表，共同数据为空；右连接相反
2.13 组合查询union
组合查询用于将相同的查询数据进行组合，比如男性的薪水UNION女性的薪水，就是所有的薪水，UNION一般直接用在两个select之间
注：UNION列类型并不需要相同，但是需要兼容；默认去重，如果不去冲，需要使用UNION ALL
2.14 全文索引(略)

3. 插入、更新、删除、创建
3.1 插入数据
例：`insert into tablename（id,name） value (10086,'中国联通')`
性能提高：实际运行时，查询服务是最重要的，所以可以通过LOW_PROIRITY来降低插入的优先级，update和delete同样
3.2 更新数据
例：`update table set id= 100 where`
3.3 删除数据
例：`delete from table where id =100`
3.4 创建和操纵表
create table tablename （
id int not null auto_increment,
name char(15) not null,
primary key (id),
）ENGINE=InnoDB;
注：每个表的自增字段只有一个，而且必须被索引
默认值指定：default 1
注：默认值只能是常量，不能使函数；开发中最好是用默认值，而不是null值
引擎类型：
	* 默认使用MyISAM：性能极高，支持全文本搜索，但是不支持事物处理
	* InnoDB可靠地事务处理，但是不支持全文本搜索
	* MEMORY功能等同于MyISAM，但是数据存储在内存中，而不是磁盘，所以适合用来存储临时表
更改表结构：`alter table tablename ADD name char(15)`
add、drop、
删除表：`drop table tablename`
重命名表：`rename table tablename1 to tablename2 
`
4. 查询辅助
4.1 使用视图
mysql5开始添加了视图功能，视图并不是一张表，而是一种虚拟表。
视图的作用：
* 重用sql，不必要每次都取写相同的语句
* 保护数据，可以只对表的部分数据给予操作，而无法操作其他的数据
* 更改表的表现形式，即重新定义表的字段
* 注意视图并不包含数据，其中的数据还是原表，只是一个原表的映射，所以，使用过多的视图联合查询，效率将很低。
* 视图不能索引，也没有关联的触发器或默认值
创建视图：create view viewname
删除视图：drop view viewname
例：`create view viewname AS select id from table1,table2 where `
4.2 使用存储过程（略）
4.3 触发器
什么是触发器：触发器相当于一个事件监听，比如我们买东西之前需要交钱，进门之前要按铃，即触发器是指在一个事件发生之前或之后触发另一个事件执行。
目前只有update、insert、delete支持触发器，其他不支持
创建触发器：`create trigger newproduct after insert on tablename for each row select 'product added'`
解析：上面创建一个名为newproduct的触发器，当tablename中插入每一行数据时，都会显示product added。
删除触发器：drop trigger triggername
后续：缺页
4.4 **事务处理**
mysql中只有InnoDB支持事务处理；
事务即一个事件，一个完整事件，有许多小步骤组成，比如我们喝茶：烧水，放茶叶，倒水等步骤，为了方式数据出现差错，这些单个的动作不能单独执行，比如如果水都没烧开，还是照样执行后面的倒水语句，那么茶基本上是喝不了，这样就出现了错误。所以事务是为了防止语句出错。即要么完全执行，要么一步都不执行。
应用场景：事务的应用场景很多，比如电商支付等
事务的几个术语：
* 事务transaction：即一组sql语句
* 回退rollback：事务中间失败，撤销之前的所有已执行语句
* 提交commit：事务完成，提交所有的语句结果到数据库
* 保留点savepoint：占位符，即用来标记回退点，不用回退整个事务





  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1533609079227.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1533609111867.jpg
