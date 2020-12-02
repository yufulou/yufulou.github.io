+++
title= "php手册里，pcntl_signal函数解释，很多例子前面declare(ticks=1)这句话的意义"
date= 2013-10-09T23:14:52+08:00
categories = ["php"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

找了好多，终于找到这个：http://www.codesky.net/article/201306/181874.html  

最后一段中：  

在此模块做模块初始化时，它会调用： php_add_tick_function(pcntl_signal_dispatch);将pcntl的分发执行函数添加到ticks机制的调用函数中去，从而当ticks触发时就会调用PCNTL扩展函数中指定的所有方法。  


所以每执行完一句话，就会dispatch一下，所以也就是注册了函数的话，就会自动去执行了