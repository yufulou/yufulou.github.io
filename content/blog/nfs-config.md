+++
title= "NFS文件系统配置，要注意一个小小问题，以及共享session的方法"
date= 2011-12-26T17:11:12+08:00
categories = ["linux"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

**<font size="4">关于NFS</font>**  
参考：http://tilt.lib.tsinghua.edu.cn/node/400（

<font size="3">Linux下NFS(网络文件系统)的建立与配置方法</font>）  

两台机器：  
一台ip：192.168.184.67  
另一台：192.168.184.71（其实无所谓，只要同网段就行…）  

192.168.184.67上设置/etc/exports：  
/home/share_dir 192.168.184.*(rw,sync,no_root_squash)    

<font color="#f00000">其实就是这里需要注意，“*”和“（”之间一定不要有空格</font>  
然后，  
/etc/init.d/portmap start  
/etc/init.d/nfs start  

另外，还需要查看系统的iptables、/etc/hosts.allow、/etc/hosts.deny是否设置了正确的NFS访问规则。  

再另一台服务器上:  
mount -t nfs 192.168.184.67:/home/share_dir /mnt -o nolock  
然后尝试修改，就会发现已经同步修改了  
如果发现mount不上，比如提示“no route to host”这种提示，则考虑主机可能开了防火墙，编辑iptables，加个例外就可以了  

配置/etc/exports时可用的参数：  
ro：只读访问   
rw：读写访问   
sync：所有数据在请求时写入共享   
async：NFS在写入数据前可以相应请求   
secure：NFS通过1024以下的安全TCP/IP端口发送   
insecure：NFS通过1024以上的端口发送   
wdelay：如果多个用户要写入NFS目录，则归组写入（默认）   
no_wdelay：如果多个用户要写入NFS目录，则立即写入，当使用async时，无需此设置。   
hide：在NFS共享目录中不共享其子目录   
no_hide：共享NFS目录的子目录   
subtree_check：如果共享/usr/bin之类的子目录时，强制NFS检查父目录的权限（默认）   
no_subtree_check：和上面相对，不检查父目录权限   
all_squash：共享文件的UID和GID映射匿名用户anonymous，适合公用目录。   
no_all_squash：保留共享文件的UID和GID（默认）   
root_squash：root用户的所有请求映射成如anonymous用户一样的权限（默认）   
no_root_squash：root用户具有根目录的完全管理访问权限   
anonuid=xxx：指定NFS服务器/etc/passwd文件中匿名用户的UID   
anongid=xxx：指定NFS服务器/etc/passwd文件中匿名用户的GID  

**<font size="4">优化：</font>**  

转自：http://icarusli.iteye.com/blog/691445（[nfs 共享session方式 session_start 慢 问题解决](http://icarusli.iteye.com/blog/691445)

）  

1.设置块大小  
mount命令的risize和wsize指定了server端和client端的传输的块大小。

mount -t nfs -o rsize=8192,wsizevb=8192,timeo=14,intr client:/partition /partition

如果未指定，系统根据nfs version来设置缺省的risize和wsize大小。大多数情况是4K对于nfs v2，最大是8K，对于v3，通过server端kernel设置risize和wsize的限制

vi /usr/src/linux2.4.22/include/linux/nfsd/const.h   
修改常量: NFSSVC_MAXBLKSIZE

所有的2.4的的client都支持最大32K的传输块。系统缺省的块可能会太大或者太小，这主要取决于你的kernel和你的网卡，太大或者太小都有可能导致nfs速度很慢。  
具体的可以使用Bonnie，Bonnie++，iozone等benchmark来测试不同risize和wsize下nfs的速度。当然，也可以使用dd来测试。 

＃time dd if=/dev/zero of=/testfs/testfile bs=8k count=1024　　测试nfs写   
＃time dd if=/testfs/testfile of=/dev/null bs=8k　　　　　　　 测试nfs读 

测试时文件的大小至少是系统RAM的两倍，每次测试都使用umount 和mount对/testfs进行挂载，通过比较不同的块大小，得到优化的块大小。

2.网络传输包的大小  
网络在包传输过程，对包要进行分组，过大或者过小都不能很好的利用网络的带宽，所以对网络要进行测试和调优。
可以使用ping -s 2048 -f hostname进行ping，尝试不同的package 
size，这样可以看到包的丢失情况。同时，可以使用nfsstat -o net 
测试nfs使用udp传输时丢包的多少。因为统计不能清零，所以要先运行此命令记住该值，然后可以再次运行统计。如果，经过上面的统计丢包很多。那么可以
看看网络传输包的大小。使用下面的命令：

\#tracepath node1/端口号   
\#ifconfig eth0

比较网卡的mtu和刚刚的pmtu，使用#ifconfig eth0 mtu 16436设置网卡的mtu和测试的一致。 
当然如果risize和wsize比mtu的值大，那么的话，server端的包传到client端就要进行重组，这是要消耗client端的cpu资
源。此外，包重组可能导致网络的不可信和丢包，任何的丢包都会是的rpc请求重新传输，rpc请求的重传有会导致超时，严重降低nfs的性能。  
可以通过查看

/proc/sys/net/ipv4/ipfrag_high_thresh  
/proc/sys/net/ipv4/ipfrag_low_thresh

了解系统可以处理的包的数目，如果网络包到达了ipfrag_high_thresh，那么系统就会开始丢包，直到包的数目到达ipfrag_low_thresh。

3.nfs挂载的优化   
timeo：　　如果超时，客户端等待的时间，以十分之一秒计算  
retrans：　超时尝试的次数。   
bg：　　　 后台挂载，很有用   
hard：　　 如果server端没有响应，那么客户端一直尝试挂载  
wsize：　　写块大小   
rsize：　　读块大小   
intr：　　 可以中断不成功的挂载   
noatime：　不更新文件的inode访问时间，可以提高速度  
async：　　异步读写

4.nfsd的个数  
缺省的系统在启动时，有8个nfsd进程   
\#ps -efl|grep nfsd  
通过查看/proc/net/rpc/nfsd文件的th行，第一个是nfsd的个数，后十个是线程是用的时间数，第二个到第四个值如果很大，那么就需要增加nfsd的个数。   
具体如下： 

\#vi /etc/init.d/nfs

找到RPCNFSDCOUNT,修改该值，一般和client端数目一致。 

\#service nfs restart   
\#mount -a 

5.nfsd的队列长度   
对于8个nfsd进程，系统的nfsd队列长度是64k大小，如果是多于8个，就要相应的增加相应的队列大小，具体的在

/proc/sys/net/core/rwmem_default  
/proc/sys/net/core/wwmem_default  
/proc/sys/net/core/rmmem_max  
/proc/sys/net/core/wmmem_max

队列的长度最好是每一个nfsd有8k的大小。这样，server端就可以对client的请求作排队处理。如果要永久更改此值 

\#vi /etc/sysctl.conf  
net.core.rmmem_default=数目  
net.core.wmmem_default=数目   
net.core.rmmem_max=数目   
net.core.wmmem_max=数目  
\#service nfs restart

++  
查看被导出资源  
showmount -e nfsserver_name(or nfsserver IP address)  
重新加载配置：  
exportfs -rv  
停止现在发布的目录  
exportfs -a  


**<font size="4">关于SESSION共享</font>**  
不过由于nfs的锁机制，在高并发情况下会有效率问题，比如想用作共享session的目录的话，可能nfs会变成性能的瓶颈，这里，有这么篇文章，写得很具体，基本就是分文件夹存储的形式：  
http://www.desteps.com/program/php/650.html（PHP实现多服务器session共享之NFS共享）  

这个帖子介绍了4中常用的session共享机制：http://www.cnblogs.com/cyw080/archive/2009/11/17/1604272.html（[四种多服务器共享session的方法](http://www.cnblogs.com/cyw080/archive/2009/11/17/1604272.html)）  
主要即：  
**1. 基于NFS的Session共享**

**2. 基于数据库的Session共享**

**3. 基于Cookie的Session共享**

**4. 基于Memcache的Session共享**

****nfs的瓶颈也许就在nfs锁机制上，数据库想必是网络传输和磁盘io，********cookie也许会存在一些安全问题，否则就是会需要加密解密的机制，耗费cpu和网络传输，********四种最快的想必是memcache的共享，并且没有安全问题****