+++
title= "sysctl.conf优化"
date= 2015-07-02T16:31:49+08:00
categories = ["linux"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

$ cat /etc/sysctl.conf  
```
# Kernel sysctl configuration file for Red Hat Linux  

#  

# For binary values, 0 is disabled, 1 is enabled.  See sysctl(8) and  

# sysctl.conf(5) for more details.  

# Controls IP packet forwarding  

net.ipv4.ip_forward = 0  

# Controls source route verification  

net.ipv4.conf.default.rp_filter = 1  

# Do not accept source routing  

net.ipv4.conf.default.accept_source_route = 0  

# Controls the System Request debugging functionality of the kernel  

kernel.sysrq = 0  

# Controls whether core dumps will append the PID to the core filename  

# Useful for debugging multi-threaded applications  

kernel.core_uses_pid = 1  

# Controls the use of TCP syncookies  

# Controls the maximum size of a message, in bytes  

kernel.msgmnb = 65536  

# Controls the default maxmimum size of a mesage queue  

kernel.msgmax = 65536  

# Controls the maximum shared segment size, in bytes  

kernel.shmmax = 68719476736  

# Controls the maximum number of shared memory segments, in pages  

kernel.shmall = 4294967296  

vm.swappiness=10  

#Randy  

net.core.somaxconn = 204800  

net.ipv4.tcp_tw_reuse = 1  

net.ipv4.tcp_tw_recycle = 0  

net.ipv4.tcp_max_orphans = 262144  

fs.file-max = 1280000  

net.ipv4.tcp_max_tw_buckets = 180000  

net.ipv4.tcp_syncookies = 1  

net.unix.max_dgram_qlen = 10000  

net.ipv4.conf.default.rp_filter = 0  

net.ipv4.tcp_max_syn_backlog = 8192  

net.ipv4.tcp_mem = 196608 262144 786432  
```

另一个版本：  
```
# Kernel sysctl configuration file for Red Hat Linux  

#  

# For binary values, 0 is disabled, 1 is enabled.  See sysctl(8) and  

# sysctl.conf(5) for more details.  

# Controls IP packet forwarding  

net.ipv4.ip_forward = 0  

# Controls source route verification  

net.ipv4.conf.default.rp_filter = 1  

# Do not accept source routing  

net.ipv4.conf.default.accept_source_route = 0  

# Controls the System Request debugging functionality of the kernel  

kernel.sysrq = 0  

# Controls whether core dumps will append the PID to the core filename  

# Useful for debugging multi-threaded applications  

kernel.core_uses_pid = 1  

# Controls the use of TCP syncookies  

# Controls the maximum size of a message, in bytes  

kernel.msgmnb = 65536  

# Controls the default maxmimum size of a mesage queue  

kernel.msgmax = 65536  

# Controls the maximum shared segment size, in bytes  

kernel.shmmax = 68719476736  

# Controls the maximum number of shared memory segments, in pages  

kernel.shmall = 4294967296  

vm.swappiness=10  

net.ipv4.tcp_keepalive_time = 300  

net.ipv4.tcp_keepalive_intvl = 10  

net.ipv4.tcp_keepalive_probes = 5  

net.ipv4.tcp_tw_reuse = 1  

net.ipv4.tcp_tw_recycle = 1  

net.ipv4.tcp_window_scaling = 0  

net.ipv4.tcp_timestamps = 0  

net.ipv4.ip_local_port_range = 1024 65535  

net.core.rmem_default = 16777216  

net.core.wmem_default = 16777216  

net.core.rmem_max = 16777216  

net.core.wmem_max = 16777216  

net.ipv4.tcp_rmem = 4096 87380 16777216  

net.ipv4.tcp_wmem = 4096 65536 16777216  

net.ipv4.tcp_fin_timeout = 10  

net.core.somaxconn = 262144  

net.ipv4.tcp_max_orphans = 262144  

net.ipv4.tcp_max_tw_buckets = 180000  

net.ipv4.tcp_no_metrics_save = 1  

net.ipv4.tcp_max_syn_backlog = 262144  

net.core.netdev_max_backlog = 300000  

net.ipv4.tcp_synack_retries = 2  

net.ipv4.tcp_syn_retries = 2  

fs.file-max = 999999  

net.ipv4.tcp_syncookies = 1  

net.unix.max_dgram_qlen = 10000  

net.ipv4.tcp_mem = 196608 262144 786432
```