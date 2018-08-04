---
title: mysql数据库进阶-leetcode-10道
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
方法一：先列出所有的分数不重复集，这样总的排位数就清楚了。然后将order后的数据与集合比较计数，>=的即为对应排位数
答案：`SELECT Score, (SELECT COUNT(DISTINCT Score) FROM Scores WHERE Score >= S.Score) AS Rank FROM Scores AS S ORDER BY Score DESC`

方法二：
考察：mysql变量赋值及初始化；mysql变量赋值使用:=，变量需要以@开头
答案：`SELECT Score,
@rank := @rank + (@pre <> (@pre := Score)) Rank
FROM Scores, (SELECT @rank := 0, @pre := -1) INIT 
ORDER BY Score DESC;`
注：两个方法逻辑类似，方法二是先初始化变量，然后计算不同值，实现累加（<>表示不等于）

### 180. Consecutive Numbers
解析：查找连续出现三次的数据
答案：`SELECT DISTINCT l1.`Num` AS ConsecutiveNums FROM LOGS l1,LOGS l2,LOGS l3 WHERE l1.Id = l2.Id - 1 AND l2.`Id` = l3.`Id`-1 AND l1.Num=l2.Num AND l2.Num=l3.Num`

### 181. Employees Earning More Than Their Managers
解析：找出满足以下雇员的工资：雇员的工资比雇主高（数据在同一张表中）
答案：`SELECT E.Name AS Employee FROM Employee AS E WHERE (SELECT Salary FROM Employee WHERE Id = E.ManagerId) < E.Salary `

### 182. Duplicate Emails
解析：查找所有重复邮件的邮件名
考察：group by和having条件；
答案：`SELECT Email FROM Person GROUP BY Email HAVING COUNT(Email) > 1`
注：这里的cout是记录没组的数量，而不是多少组
注2：having和where的区别：group中有很多聚合函数，比如求均值AVG等，此时聚合函数不能和where联合使用

### 183. Customers Who Never Order
解析：查找从未购买过东西的用户。可以直接使用左连接，获取右表为null的数据
考察：左连接
答案：`SELECT `Name` AS Customers FROM Customers LEFT JOIN orders ON Customers.Id = Orders.CustomerId WHERE Orders.CustomerId IS NULL`

### 184. Department Highest Salary
解析：查找每个部门最高工资的员工数据
考察：联合查找
标准答案：`SELECT
    Department.name AS 'Department',
    Employee.name AS 'Employee',
    Salary
FROM
    Employee
        JOIN
    Department ON Employee.DepartmentId = Department.Id
WHERE
    (Employee.DepartmentId , Salary) IN
    (   SELECT
            DepartmentId, MAX(Salary)
        FROM
            Employee
        GROUP BY DepartmentId
    )`
我的答案：`SELECT D.Name AS Department, E.Name AS Employee,MAX(E.Salary) AS Salary  FROM Employee AS E LEFT JOIN Department AS D ON D.Id = E.DepartmentId GROUP BY D.Name` 
注：我的答案是按部门排序的，题目需要按照id排序

### 185. Department Top Three Salaries
解析：查找每个部分工资最高的3个员工数据。
难点：这题有点难，需要两个条件：首先需要通过`WHERE 3 > (SELECT COUNT(DISTINCT(e2.Salary)) 
                  FROM Employee e2 
                  WHERE e2.Salary > e1.Salary 
                  AND e1.DepartmentId = e2.DepartmentId
                  );`筛选出最高的三个员工薪资行，大于3的不要；
答案：`SELECT d.Name Department, e1.Name Employee, e1.Salary
FROM Employee e1 
JOIN Department d
ON e1.DepartmentId = d.Id
WHERE 3 > (SELECT COUNT(DISTINCT(e2.Salary)) 
                  FROM Employee e2 
                  WHERE e2.Salary > e1.Salary 
                  AND e1.DepartmentId = e2.DepartmentId
                  );`

	
  [1]: https://www.cnblogs.com/Evil-Rebe/p/5885649.html
