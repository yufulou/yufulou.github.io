+++
title= "[转]在MySQL中，如何计算一组数据的中位数？"
date= 2021-01-07T14:20:38+08:00
categories = ["mysql", "sql"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

原文链接地址：https://learnku.com/articles/18390

要得到一组数据的中位数（例如某个地区或某家公司的收入中位数），我们一般要将这一任务细分为 3 个小任务：

1.  将数据排序，并给每一行数据给出其在所有数据中的排名；
2.  找出中位数的排名数字；
3.  找出中间排名对应的值；

下面以某公司员工月收入为例，示例 MySQL 的一些复杂语句的使用。

## 方法一

#### 创建测试表

首先创建一个收入表，建表语句为：

```
CREATE TABLE IF NOT EXISTS `employee` (
  `id`     INT                  AUTO_INCREMENT PRIMARY KEY,
  `name`   VARCHAR(10) NOT NULL DEFAULT '',
  `income` INT         NOT NULL DEFAULT '0'
)
  ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

INSERT INTO `employee` (`name`, `income`)
VALUES ('麻子', 20000);
INSERT INTO `employee` (`name`, `income`)
VALUES ('李四', 12000);
INSERT INTO `employee` (`name`, `income`)
VALUES ('张三', 10000);
INSERT INTO `employee` (`name`, `income`)
VALUES ('王二', 16000);
INSERT INTO `employee` (`name`, `income`)
VALUES ('土豪', 40000);
```


#### 完成任务 1

将数据排序，并给每一行数据给出其在所有数据中的排名：

```
SELECT t1.name, t1.income, COUNT(*) AS rank
FROM employee AS t1,
     employee AS t2
WHERE t1.income < t2.income
   OR (t1.income = t2.income AND t1.name <= t2.name)
GROUP BY t1.name, t1.income
ORDER BY rank;
```

查询结果为：

name | income | rank
---- | ------ | ----
土豪   | 40000  | 1   
麻子   | 20000  | 2   
王二   | 16000  | 3   
李四   | 12000  | 4   
张三   | 10000  | 5   

#### 完成小任务 2

找出中位数的排名数字：

```
SELECT (COUNT(*) + 1) DIV 2 as rank
FROM employee;
```

<button class="copy-code-button ui label" style="position: absolute; top: 20px; right: 1px; display: none;">Copy</button>

查询结果为：

| rank |
| ---- |
| 3 |

#### 完成小任务 3

```
SELECT income AS median
FROM (SELECT t1.name, t1.income, COUNT(*) AS rank
      FROM employee AS t1,
           employee AS t2
      WHERE t1.income < t2.income
         OR (t1.income = t2.income AND t1.name <= t2.name)
      GROUP BY t1.name, t1.income
      ORDER BY rank) t3
WHERE rank = (SELECT (COUNT(*) + 1) DIV 2 FROM employee)
```


查询结果为：

| median |
| ------ |
| 16000 |

至此，我们就找到了如何从一组数据中获得中位数的方法。

## 方法二

下面，来介绍另外一种优化排名语句的方法。

我们都知道如何给一组数据做排序操作，在本例中，实现方法如下：

```
SELECT name, income
FROM employee
ORDER BY income DESC
```


查询结果为：

name | income
---- | ------
土豪   | 40000 
麻子   | 20000 
王二   | 16000 
李四   | 12000 
张三   | 10000 

那我们可不可以更进一步，对查询出的结果加一列，这一列的数据为排名呢？

我们可以通过 3 个自定义变量的方法来实现这一目标：

第一个变量用来记录当前行数据的收入  
第二个变量用来记录上一行数据的收入  
第三个变量用来记录当前行数据的排名

```
SET @curr_income := 0;
SET @prev_income := 0;
SET @rank := 0;

SELECT `name`,
       @curr_income := income                                      AS income,
       @rank := if(@prev_income != @curr_income, @rank + 1, @rank) AS rank,
       @prev_income := @curr_income                                AS dummy
FROM employee
ORDER BY income DESC
```

查询结果如下：

name | income | rank | dummy
---- | ------ | ---- | -----
土豪   | 40000  | 1    | 40000
麻子   | 20000  | 2    | 20000
王二   | 16000  | 3    | 16000
李四   | 12000  | 4    | 12000
张三   | 10000  | 5    | 10000

然后再找出中位数的排名数字，进一步找出收入的中位数：

```
SET @curr_income := 0;
SET @prev_income := 0;
SET @rank := 0;

SELECT income AS median
FROM (SELECT `name`,
             @curr_income := income                                      AS income,
             @rank := if(@prev_income != @curr_income, @rank + 1, @rank) AS rank,
             @prev_income := @curr_income                                AS dummy
      FROM employee
      ORDER BY income DESC) AS t1
WHERE t1.rank = (SELECT (COUNT(*) + 1) DIV 2 FROM employee)
```


查询结果为：

| median |
| ------ |
| 16000 |

至此，我们找了两种方法来解决中位数的问题。撒花。

> 本作品采用[《CC 协议》](https://learnku.com/docs/guide/cc4.0/6589)，转载必须注明作者和本文链接