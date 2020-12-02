+++
title= "VMware装mac虚拟机时候需要注意的"
date= 2014-08-19T11:29:08+08:00
categories = ["op"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

1、要是开机出磁盘。。。unsuccessful什么的，打开vmx文件，删了efi那一行（换个iso后来就没这事了）
2、要是就卡在读磁盘读不出来那块了，有两种可能。a、iso有问题，b、做虚拟机时候，需要磁盘文件是一个文件，不能分到多个文件（可能需要打开显示器的3D加速）
3、装好以后，把cd启动盘换成darwin.iso