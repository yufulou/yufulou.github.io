+++
title= "一个通过smtp发送邮件的shell，带用户名密码"
date= 2012-01-06T16:55:14+08:00
categories = ["php"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

```
\#!/bin/bash  

smail(){  
        smtp="mail.mailadd.com 25" # 邮件服务器地址+25端口  
        smtp_domain="mailadd.com" # 发送邮件的域名，即@后面的  
        FROM="xxx@mailadd.com" # 发送邮件地址  
        RCPTTO=$1 # 收件人地址  
        username_base64="xxxxxxxxxxxxxxxxx" # 用户名base64编码  
        password_base64="xxxxxxxx" # 密码base64编码  
        local_ip=`ifconfig|grep Bcast|awk -F: '{print $2}'|awk -F " " '{print $1}'|head -1`  
        local_name=`uname -n`  
        ( for i in "ehlo $smtp_domain" "AUTH LOGIN" "$username_base64" "$password_base64" "MAIL FROM:<$FROM>" "RCPT TO:<$RCPTTO>" "DATA";do  
                echo $i  
                sleep 4  
        done  
        echo "Subject:server alert"  
        echo "From:<$FROM>"  
        echo "To:<$RCPTTO>"  
        echo "server $local_name up, ip:$local_ip"  
        echo "."  
        sleep 2  
        echo "quit" )|telnet $smtp  
}  
smail xxx@163.com # 这里参数为收信地址
```