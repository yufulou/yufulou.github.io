+++
title= "redis可能波动的几种情况"
date= 2018-09-29T17:25:07+08:00
categories = ["redis"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

内存占用太大时，bgsave的fork复制内存

aof文件fsync时间过长

内存碎片太多，导致重新布局

不常用的value入swap，导致读取变慢