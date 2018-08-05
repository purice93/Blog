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

### 

### 

### 

### 

### 

