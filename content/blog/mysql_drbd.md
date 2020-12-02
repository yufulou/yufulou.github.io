+++
title= "redhat5.x86_64下搭建mysql drbd heartbeat做主从mysql高可用（mysql HA）"
date= 2011-12-26T15:39:54+08:00
categories = ["linux"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

参考：http://www.boobooke.com/bbs/thread-47791-1-1.html（<font size="3">Heartbeat   Drbd  Mysql 构建高可用的MYSQL数据库服务</font>）  

redhat1(master):172.31.1.73  
redhat2(slave):172.31.1.71  
VIP:172.31.1.77  

**<font size="4">一、准备工作</font>**  
1、保证有未格式化的，或者未分区的磁盘空间  
2、修改两机hostname：  
 1）两机修改：/etc/hosts文件，加入如下2行（使服务器可通过别名访问）  
     172.31.1.71   slave  
     172.31.1.73   master  
 2）两机修改: /etc/sysconfig/network，把HOSTNAME改为master或者slave  
 3）两机执行：hostname master 或者 hostname slave  
 4）两机测试：ping master；ping slave，若都能ping通，则说明设置正确  
3、更新源并yum update  
4、安装mysql（我安装的是64位的二进制版），不设置开机启动  

**<font size="4">二、安装并配置drbd</font>**  
1、两机执行：yum -y install drbd83 kmod-drbd83  
   安装后，执行lsmod | grep drbd  
   有类似如下显示，则证明安装成功  
   drbd                  228528  3  
   如果发现正常安装无法加载drbd模块，则要通过"uname -a"检查系统是否处于xen内核模式下，如果是，修改/boot/grub/grub.conf，修改默认启动内核为不带"xen"版本的  
2、两机修改：/etc/drbd.conf  
```
   \#  
   \# please have a a look at the example configuration file in  
   \# /usr/share/doc/drbd83/drbd.conf  
   \#  
   global { usage-count yes; }  
   common { syncer { rate 10M; } }  
   resource r0{  
    protocol C;  
    startup{  
           degr-wfc-timeout 120;  
    }  
    disk{  
           on-io-error detach;  
    }  
    net{  
    }  
    syncer{  
           rate 10M;  
    }  
    on master{  
           device          /dev/drbd0;  
           disk            /dev/vdb1; // 因情况而异  
           address         172.31.1.73:7788;    
           meta-disk       internal;  
    }  
    on slave{  
           device          /dev/drbd0;  
           disk            /dev/vdb1; // 因情况而异  
           address         172.31.1.71:7788;  
           meta-disk       internal;  
    }  
   }  
  // 再创建一个资源，为二进制日志做准备  
   resource r1{  
    protocol C;  
    startup{  
           degr-wfc-timeout 120;  
    }  
    disk{  
           on-io-error detach;  
    }  
    net{  
    }  
    syncer{  
           rate 10M;  
    }  
    on master{  
           device          /dev/drbd1;  
           disk            /dev/vdb2; // 因情况而异  
           address         172.31.1.73:7789; // 端口号需不同，否则报错    
           meta-disk       internal;  
    }  
    on slave{  
           device          /dev/drbd1;  
           disk            /dev/vdb2; // 因情况而异  
           address         172.31.1.71:7789; // 端口号需不同，否则报错   
           meta-disk       internal;  
    }  
   }  
```
3、两机执行：for i in $(seq 0 15) ; do mknod /dev/drbd$i b 147 $i ; done  
   建立设备文件，不过貌似有drbd0就够了，上面的语句会创建0-15  
4、初始化meta-data area  
   两机执行drbdadm create-md r0, drbdadm create-md r1  
   如果这步出了error，很有可能是配置中指定的分区已经被格式化过，或曾经格式化过（我删除分区重建也不行），则执行：dd if=/dev/zero bs=1M count=1 of=/dev/vdb1; sync，然后重新执行drbdadm create-md r0, drbdadm create-md r1  
5、两台主机执行：/etc/init.d/drbd start;   
   1）测试：netstat -ant | grep $PORT:7788，显示如下，则正确：  
      tcp     0   0 172.31.1.73:7788      0.0.0.0:*       ESTABLISHED  
   2）cat /proc/drbd，会发现如Secondary/Secondary字样，证明两台都是备机状态  
   3）在相当做主机的设备上执行：  
      drbdsetup /dev/drbd0 primary -o，drbdsetup /dev/drbd1 primary -o  
      cat /proc/drbd，则会发现正在执行同步，并且状态变为Primary/Secondary  
6、同步完成后，则可开始对drbd分区任意格式化（从这里开始，一切对/dev/vdb1的操作，都变为对/dev/drbd0操作），如"mkfs.ext3 /dev/drbd0"，  
   然后可以"mount /dev/drbd0 /home/Db/","mount /dev/drbd1 /home/log"  
7、可以开始测试，并写入/home/Db分区，在另一边看是否可以同步，在备机查看时，要先停掉drbd服务（service drbd stop），然后"mount /dev/vdb1 /home/Db"，看是否同步  
8、两机主备切换命令：drbdadm secondary r0  ； drbdadm primary r0 ；前提是，要先把主机drbd停止或降级，然后备机升为primary  
9、配置mysql数据目录为/home/Db/mysql  

<font size="4"><b>三、安装配置heartbeat</b></font>（均为两机执行）  
1、yum install -y heartbeat(heartbeat可能需要安装2次，暂时是个神奇的问题……)  
2、cp /usr/share/doc/heartbeat-2.1.3/haresources /etc/ha.d/  
   cp /usr/share/doc/heartbeat-2.1.3/ha.cf /etc/ha.d/  
   cp /usr/share/doc/heartbeat-2.1.3/authkeys /etc/ha.d/  
   chmod 600 /etc/ha.d/authkeys（注意，这里必须要为600，否则会报错）  
3、vi /etc/ha.d/authkeys，去掉以下2行注释  
   auth 1  
   1 crc  
4、vi /etc/ha.d/ha.cf，改后为  
```
   \#grep -v "^#" /etc/ha.d/ha.cf  
   debugfile /var/log/ha-debug  
   logfile /var/log/ha-log  
   logfacility     local0  
   keepalive 2  
   deadtime 10  
   warntime 5  
   initdead 120  
   ucast eth0 172.31.1.71   # 互填对方ip，这里slave写172.31.1.73  

   auto_failback off    # 恢复正常后是否需要再自动切换回来，一般都设为off。  
   node    master  
   node    slave  
   ping 172.31.1.67  # 这里放一个两台服务器均应能ping通的第三方节点IP即可，目的是让服务器能自知与外界的连通性  
   \# 指定和heartbeat一起启动、关闭的进程  
   respawn hacluster /usr/lib64/heartbeat/ipfail   
   apiauth ipfail gid=haclient uid=hacluster  
```
5、vi /etc/ha.d/resource.d/mysqld_umount（设置heartbeat主从转换时执行的脚本）  
```
   \#!/bin/bash  
   unset LC_ALL; export LC_ALL  
   unset LANGUAGE; export LANGUAGE  

   prefix=/usr  
   exec_prefix=/usr  
   . /etc/ha.d/shellfuncs  

   case "$1" in  
   'start')  
           drbdadm primary all  
           mount /dev/drbd0 /home/Db  
           mount /dev/drbd1 /home/log  
           drbdadm connect all  
   ;;  
   'pre-start')  
   ;;  
   'post-start')  
   ;;  
   'stop')  
           umount /dev/drbd1  
           umount /dev/drbd0  
           drbdadm secondary all  
   ;;  
   'pre-stop')  
   ;;  
   'post-stop')  
   ;;  
   \*)  
           echo "Usage: $0 { start | pre_start | post-start | stop | pre-stop |post-stop }"  
   ;;  

   esac  
   exit 0  
```
6、chmod  x  /etc/ha.d/resource.d/mysqld_umount  
7、vi /etc/ha.d/haresource 添加 "master  IPaddr::172.31.1.77/24/eth0:1 mysqld_umount mysql"  
8、service heartbeat start，先启动master的heartbeat服务，然后启动slave的。  
9、测试  
10、设置drbd自动加载，但mysql不自动加载，heartbeat写在rc.local中，保证在drbd之后启动  

**<font size="4">四、mysql二进制日志及redo日志设置</font>**（摘自mysqlops朋友的“Mysql服务器端参数详解和优化建议.pdf”）  
innodb_flush_log_at_trx_commit= N：  
N=0 --每隔一秒，把事务日志缓存区的数据写到日志文件中，以及把日志文件的数据刷新到磁盘上；  
N=1 –每个事务提交时候，把事务日志从缓存区写到日志文件中，并且刷新日志文件的数据到磁盘上；  
N=2 –每事务提交的时候，把事务日志数据从缓存区写到日志文件中；每隔一秒，刷新一次日志文件，但不一定刷新到磁盘上，而是取决于操作系统的调度；  
sync_binlog = N：  
N>0 --每向二进制日志文件写入N条SQL或N个事务后，则把二进制日志文件的数据刷新到磁盘上；  
N=0 --不主动刷新二进制日志文件的数据到磁盘上，而是由操作系统决定；  
推荐配置组合：  
N=1,1 ---适合数据安全性要求非常高，而且磁盘IO写能力足够支持业务，比如充值消费系统；  
N=1,0 ---适合数据安全性要求高，磁盘IO写能力支持业务不富余，允许备库落后或无复制；  
N=2,0或2,m(0

<m<100) ---适合数据安全性有要求，允许丢失一点事务日志，复制架构的延迟也能接受；<br="">N=0,0 ---磁盘IO写能力有限，无复制或允许复制延迟稍微长点能接受，例如：日志性登记业务</m<100)>