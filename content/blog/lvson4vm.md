+++
title= "【原】vmware上4台centos5.7.i386做2*LVS/DR(master backup) 2*realserver"
date= 2011-12-23T17:57:21+08:00
categories = ["lvs"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

参考的是这篇文档：http://www.360doc.com/content/11/0301/10/834950_97080013.shtml  
但该文档有些错误，所以自己又按自己的流程整理了下，整体没啥变化，所以应该能用到实际生产中（虽然具体，但只进行了初步测试，应该会有一些问题，只是为以后自己用时，做个记录）：  
**<font size="4">前提操作：</font>**  
1、设置源（我这个默认那个源就很给力，所以没设置）  
2、yum update  
3、yum安装kernel-headers,kernel-devel  
4、yum安装httpd、php、mysql等服务（php、mysql为测试用）  
5、yum安装gcc，g  ，openssl-devel  
6、vi /etc/sysctl.conf 修改net.ipv4.ip_forward=1  
7、编辑/etc/inittab，  
   把“id:5:initdefault:”改成“id:3:initdefault:”，  
   关了图形界面，开关机快些、占用资源少些  
8、然后，关掉虚拟机（先关机，然后从标签页中，按X，彻底关掉），彻底关掉前，可以调下内存，事实证明，我把4个虚拟机都调成256M，运行流畅，也许是关了图形界面的缘故也没准……  
9、又是图省事：复制虚拟机文件夹为centos1，centos2，centos3，centos4，并重新通过vmware加载，改标签为leader，backup，rs1，rs2，好认，在里面也改hostname为这几个。  
10、VIP：192.168.184.138  
   lvs1(leader):192.168.184.139  
   lvs2(backup):192.168.184.140  
   rs1: 192.168.184.141  
   rs2: 192.168.184.142  
**<font size="4">正式操作：</font>**  
1、主备LVS：yum install ipvsadm  
2、主备LVS：下载并安装keepalived-1.2.2.tar.gz  
   1)需安装openssl-devel  
   2)./configure --prefix=/usr/local/keepalived  
   3)make&&make install  
   4)cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/  
   5)cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/  
   6)mkdir /etc/keepalived  
   7)cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/  
   8)cp /usr/local/keepalived/sbin/keepalived /sbin/  
3、主备LVS：建立负载脚本  
   vim /sbin/lvsdr.sh  
```
   \#!/bin/bash  

   VIP=192.168.184.138  
   RIP1=192.168.184.141  
   RIP2=192.168.184.142  
   /etc/rc.d/init.d/functions  
   case "$1" in  
      start)  
        echo "start LVS of DirectorServer"  
        \#Set th Virtual IP Address  
        ifconfig eth0:1 $VIP broadcast $VIP netmask 255.255.255.255 up  
        route add -host $VIP dev eth0:1  
        \#Clear IPVS Table  
        ipvsadm -C  
        \#Set Lvs  
        ipvsadm -A -t $VIP:80 -s wrr  
        ipvsadm -a -t $VIP:80 -r $RIP1:80 -g  
        ipvsadm -a -t $VIP:80 -r $RIP2:80 -g  
        \#Run LVS  
        ipvsadm  
      ;;  
      stop)  
        echo "Close LVS Director server"  
        ifconfig eth0:1 down  
        ipvsadm -C  
      ;;  
      \*)  
        echo "Usage:{start|stop}"  
        exit 1  
   esac  
```
4、主备LVS：chown 755 /sbin/lvsdr.sh  
5、测试：  
   lvsdr.sh start:  
```
      ifconfig出现eth0:1 XXX  
      route -n中destination多了vip就对了  
```
   lvsdr.sh stop:  
      与上面完全相反  
6、两台realserver：  
   vim /sbin/realdr.sh  
```
   \#!/bin/bash  
   VIP=192.168.184.138  
   ifconfig lo:0 $VIP broadcast $VIP netmask 255.255.255.255 up  
   route add -host $VIP dev lo:0  
   echo "1" > /proc/sys/net/ipv4/conf/default/arp_ignore  
   echo "2" > /proc/sys/net/ipv4/conf/default/arp_announce  
   echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore  
   echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce  
   sysctl -p  
```
7、两台realserver：chmod 755 /sbin/realdr.sh  
8、两台realserver：分别执行/sbin/realdr.sh  
9、主备LVS：编辑/etc/keepalived/keepalived.conf:  
```
   !Configuration File for keepalived  
   gloabl_defs{  
           notification_email{  
                   yufulou@qq.com  
           }  
           notification_email_from yu.guo@echance.cn  
           smtp_server 192.168.200.20  
           smtp_connect_timeout 30  
           router_id LVS_DEVEL  
   }  
   vrrp_instance VI_1{  
           state MASTER // 从服务器设置BACKUP  
           interface eth0  
           virtual_router_id 51  
           priority 100 // 从服务器设置比100小的值  
           advert_int 1  
           authentication(  
                   auth_type PASS  
                   auth_pass 1111  
           )  
           virtual_ipaddress{  
                   192.168.184.138  
           }  
   }  
   virtual_server 192.168.184.138 80{  
           delay_loop 6  
           lb_algo wrr  
           lb_kind DR  
           persistence_timeout 60  
           inhibit_on_failure  
           protocal TCP  
           real_server 192.168.184.141 80{  
                   weight 3  
                   TCP_CHECK{  
                           connect_timeout 10  
                           nb_get_retry 3  
                           delay_before_retry 3  
                   }  
           }  
           real_server 192.168.184.142 80{  
                   weight 3  
                   TCP_CHECK{  
                           connect_timeout 10  
                           nb_get_retry 3  
                           delay_before_retry 3  
                   }  
           }  
   }  
```
10、主备LVS：/etc/init.d/keepalived start  
11、主备LVS：  
```
   echo "/etc/init.d/keepalived restart" >> /etc/rc.local  
   echo "/sbin/lvsdr.sh" >> /etc/rc.local  
```
12、两台realserver:  
```
   echo "/sbin/realdr.sh" >> /etc/rc.local  
```
13、测试  
