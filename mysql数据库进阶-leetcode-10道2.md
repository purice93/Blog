---
title: mysql数据库进阶-leetcode-10道2
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
### 196. Delete Duplicate Emails
解析：删除重复值邮箱
答案：`DELETE P1 FROM Person p1,Person P2 WHERE P1.Email = P2.Email AND P1.Id > P2.Id`

### 197. Rising Temperature
解析：查找温度相对于前一天上升的数据；需要使用sql日期处理函数
考察：sql的日期处理函数**TO_DAYS**使用
答案：`SELECT W1.Id FROM Weather W1,Weather W2 WHERE W1.Temperature > W2.Temperature AND TO_DAYS(W1.RecordDate) - TO_DAYS(W2.RecordDate) = 1`

### 262. Trips and Users
解析：查找旅行数据中，每日订单的取消率；条件：1时间期限、2用户没有罚款
考察：联合查询、between、round函数、if函数
步骤：先查数据，然后向其中添加条件
答案：`Write a SQL query to find the cancellation rate of requests made by unbanned users between Oct 1, 2013 and Oct 3, 2013. For the above tables, your SQL query should return the following rows with the cancellation rate being rounded to two decimal places.`

### 595. Big Countries
答案：`SELECT name, population, area from World where  area > 3000000 or population > 25000000 `

### 596. Classes More Than 5 Students
答案：`SELECT class from (SELECT class,COUNT(DISTINCT student) AS num from courses  group by class) AS table1 WHERE num >=5`

### 601. Human Traffic of Stadium
解析：对于体育场人流量数据表，查询连续三天或以上人流量大于100的数据；需要使用聚合
答案：分布书写
``` 
SELECT DISTINCT(S.id),date,people from stadium as S,(SELECT s1.id from stadium s1, stadium s2, stadium s3 WHERE  s1.people >= 100 and s2.people >= 100 and s3.people >= 100
and (s1.id = s2.id-1 and s2.id = s3.id -1) order by s1.id) AS t1 where S.id=t1.id or S.id=t1.id+1 or S.id=t1.id +2 
```

### 620. Not Boring Movies
解析：对有趣的电影进行评分排序
考点：MOD函数进行奇数判断
答案：`SELECT * FROM cinema WHERE MOD(id,2)=1 and description!='boring' order by rating DESC`

### 626. Exchange Seats
解析：交换奇偶位置数据；拼接三个表：奇数表，偶数表，最后的奇数表（总数为奇数时）
考察：联合数据UNION，拼接几个表；
答案：`SELECT s1.id+1 AS id,s1.student from seat s1 WHERE MOD(id,2)=1 and s1.id!=(SELECT MAX(id) FROM seat) 
UNION 
SELECT s2.id-1,s2.student from seat s2 WHERE MOD(id,2)=0 
UNION
SELECT s3.id,s3.student from seat s3 WHERE MOD(id,2)=1 and s3.id=(SELECT MAX(id) FROM seat) 
order by id`

### 627. Swap Salary
解析：交换性别
考察：使用if或者case，来判断条件进行赋值；不能使用临时表
答案：`UPDATE salary SET sex = CASE sex WHEN 'f' THEN 'm' ELSE 'f' END`



