+++
title= "mysql5.1.58 触发器中调用存储过程 出现“Table was not locked with LOCK TABLES”的解决办法"
date= 2012-08-29T19:02:20+08:00
categories = ["mysql"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

第一次遇到这个问题，网上也没查到相关办法，特发博客记录一手  
先定义2个临时叫法：  
1级存储过程：触发器中直接调用的存储过程  
N+1级存储过程：1级存储过程调用的子存储过程，以及子存储过程调用的存储过程。。。。。  

先大概说一下：

1、官方文档表示：如果运行了lock table，则以后对任何表进行操作前，都必须lock table，否则，就提示本文标题那个错误。

2、有个特殊情况，就是，在被lock表的触发器中，如果写了对另外一个实体表的操作，则另外那个实体表自动被加相同的锁。该原则，适用于1级及N+1级所有存储过程，所有在操作涉及到的实体表都会自动被加相同的锁，所以都不会出错。

3、但，如果被lock表的触发器的1级存储过程创建了一个临时表，则该临时表只能在本存储过程内被操作，N+1级存储过程想对其进行操作，则会提示：Table 
'xxxxx' was not locked with LOCK TABLES。

4、存储过程中不能使用lock table语句  

然后，我就发现了一个有意思的办法，就是“声明”

只要在N+1级存储过程中，这样声明一下1级存储过程创建的临时表名，就可以安全过关了：create temporary table if 
not exists xxxxx(col1 int);

此时列名、类型都不需要一样  

今天时间不够了，下次试验一下prepare statement会不会有类似问题，但估计得有  


实例  

建表：  
![image](/img/blog/mysql-trigger-error/1.jpg)

触发器：  
![image](/img/blog/mysql-trigger-error/2.jpg)  

1级存储过程直接操作：  
![image](/img/blog/mysql-trigger-error/3.jpg)

1级存储过程操作成功：  
![image](/img/blog/mysql-trigger-error/4.jpg)

N+1级存储过程不声明：  
![image](/img/blog/mysql-trigger-error/5.jpg)

N+1级存储过程调用失败：  
![image](/img/blog/mysql-trigger-error/6.jpg)

N+1级存储过程声明临时表（此时，声明不需要列名，列数，以及类型匹配）：  
![image](/img/blog/mysql-trigger-error/7.jpg)

N+1级存储过程调用成功（可以看到，此时，不严格的声明，不会对临时表结果造成影响）：  
![image](/img/blog/mysql-trigger-error/8.jpg)