---
title: mysql数据库进阶-leetcode
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
### 175. Combine Two Tables
解析：问题为联合查询两个表Person和Address的数据，无论address中是否有person对应的数据。即输出所有的person数据，address中没有的数据显示为空
考察：内连接、外连接（左连接、右连接）
内连接：join on
左连接：left on
右连接：right on
答案：
`SELECT P.FirstName, P.LastName,  A.City, A.State FROM Person AS P LEFT JOIN Address AS A ON A.PersonId = P.PersonId`
输出：`["Allen","Wang",null,null]`

### 176. Second Highest Salary
解析：问题查找一个表，返回薪资倒数第二薪资数目。
考察：limit和offset的使用，以及null的判断；多用在数据分页中；group和order区别：此处即薪资相同的算作一组数据，所以需要使用group
group：字面意思就是分组，即相同的值会聚合到一起；比如查所有的员工，并按照部分分组。
order：即只是提供排序，不聚合
两种分页方法：
limit m,n：从第m条开始读，读取n条数据（不包括第m条）
limit m offset n：从第n条开始读，读取m条数据（不包括第n条，offset为偏移量）
答案：`SELECT IFNULL((SELECT Salary FROM Employee GROUPBY Salary DESC LIMIT 1 OFFSET 1), NULL) AS SecondHighestSalary`
或者：`SELECT IFNULL((SELECT Salary FROM Employee GROUPBY Salary DESC LIMIT 1, 1), NULL) AS SecondHighestSalary`

注：这里limit有一个优化，即当偏移量很大时，可以先找出偏移量，在查找数据[参考博客][1]

### 177. Nth Highest Salary
解析：直接将N-1替换为之即可
考察：mysql语句中不能直接使用计算，即n-1，需要提前设置N的值
答案：

``` stylus
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  SET N = N-1;
  RETURN (
      # Write your MySQL query statement below.
      SELECT IFNULL((SELECT Salary FROM Employee GROUP BY Salary DESC LIMIT 1 OFFSET N), NULL) AS SecondHighestSalary
  );
END
```

### 178. Rank Scores
解析：即对学生成绩进行排位，相同分数位数一致（学校常用）
考察：

  [1]: https://www.cnblogs.com/Evil-Rebe/p/5885649.html
