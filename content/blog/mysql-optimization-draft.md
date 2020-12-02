+++
title= "随想随写的mysql性能优化流水账"
date= 2013-10-09T17:48:04+08:00
categories = ["mysql"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

总结mysql优化方式：

1、  语句方面工具：

a)         Explain

b)        Set profiling=1

2、  语句方面的注意事项：

a)         降低每次查找的数据量，避免出现需要使用tmp file的情况

b)        把in尽量写成exists，not in => not exists

c)         需要Like的场景尽量使用左匹配

d)        如果可以把两次查找放到同一个sql中，并不增加语句复杂度，尽量这样做，对速度有帮助

e)         可以在一定情况下，使用force/ignore/use index语句指定查询索引，提高速度

f)         尽量not null

g)        在字符串长度变化不大的场景下，用char替换varchar可提高查找速度

h)        复杂的查询使用procedure，减少进程间交互

i)          使用HIGH_PRIORITY提高需要尽快返回结果的查询的优先级

j)          使用SQL_CALC_FOUND_ROWS进行分页的总数统计，少用一次count，对innodb查询作用比较大

3、  提高innodb_buffer_pool_size，对innodb的插入查找速度都有帮助(innodb_log_file_size=innodb_buffer_pool_size/redo日志)

4、  提高key_buffer_size可对myisam查询很有帮助

5、  加入skip-name-resolve，避免mysql的dns反查

6、  活用几种cache/buffer(sort_buffer_size:排序缓存，存储order by的中间结果，超过这个值则会使用tmp file；read_buffer_size:循序查找缓存；read_rnd_buffer_size:随机查找缓存；query_cache:语句缓存，增大可加快对更新不快的数据表的查询；join_buffer_size:联合查找的中间缓存) – 并不是越大越好，需要有个合适的值

7、  Myisam使用concurrent_insert=2，在队尾插入，定期空闲时间optimize table，可保证查询速度的同时，不使表增大太快

8、  使用slow_query_log记录慢查询，并进行优化

9、  可定期analyze table，使mysql优化选择器选择更合适的方案

10、              开启二进制日志(可用mixed方式)，可辅助redo日志进行数据库崩溃时的修复。（可定期检查并删除过期日志避免磁盘空间增大，如果场景数据增量比较稳定，用expire_logs_days即可）

11、              分表，分区，垂直分割数据表（基本信息表、附加信息表）

12、              通过replication做主从负载均衡

13、              通过drbd+heartbeat做高可用