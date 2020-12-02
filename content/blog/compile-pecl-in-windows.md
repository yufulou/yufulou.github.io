+++
title= "compile pecl in windows"
date= 2014-02-06T19:15:37+08:00
categories = ["php"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

看了N多文章，就没有一个能顺利做下来的……你们真的都试过了吧……  

没在windows下编译过c程序，这回算是长了大经验了，耗时几十小时得有了……  

试验的是pecl_http这个扩展  

遇到的问题：（想到哪说哪，不分先后）  

1、vs2008环境变量的设置，Common7/IDE、VC/bin、Microsoft SDKs\Windows\v7.0\Bin、\win32build\bin需要设置到path  

2、nmake以前需要运行VC/vcvarsall.bat  

3、zlib版本必须大于1.2.5（尝试编译php时），否则会报compress什么的一堆东西未定义，win32build里面好多lib可能都太老了  

4、vs2008导入pecl_http后，好多.h和.c没有被导入到工程(pecl_http1.7.6里面有个工程文件)，导致编译问题  

5、calender.c和jewish.c的编码问题，只能通过editplus的西欧打开，另存为utf8解决（尝试编译php时）  

6、configure.js每次运行都TM报错，还是php官网给的方法，FXXK！这方法有待研究  

7、预处理器ZEND_DEBUG=0，这个一定要设置，否则一定报错  

8、cscript.exe configure.js --enable-http=shared，nmake以后，没找着http.dll……是官方办法，失败了……  

9、得找个libcurl.lib和openssl那几个lib，拷到win32build/lib里，然后把链接就连接到win32build/lib就行了  

10、编译出来了http.dll，放到ext目录，改php.ini，重启加载，就报错，说在那个目录下面找不到http.dll，肯定还是哪有问题，继续找错……  

11、[error C2466: 不能分配常量大小为0 的数组](http://sheng.iteye.com/blog/1228239) --- [http://sheng.iteye.com/blog/1228239  ](http://sheng.iteye.com/blog/1228239)12、把默认的_DEBUG标识换成NDEBUG，否则编译时会有“error LNK2001: unresolved external symbol __imp___free_dbg”的问题(http://www.jdelist.com/ubb/showflat.php?Cat=&Number=92220&page=0&view=collapsed&sb=5&o=)  

13、一开始链接到一个64位的php的dev的php5ts.lib上，一编译，报了516个错；换成了32位的，瞬间只剩了27个错（不确定是不是真的就是这个原因），最后成功编译也是链接到32位的php5ts.lib上成功的，下次可以试试64位的。（PS：自己编译了一个32位的php，效果和下载的一样）  

14、打开pecl_http的工程文件，里面默认是需要libcurl(带SSL版本)和zlib的头和lib的，都需要下载下来，设置到工程里  

15、一定还有什么别的问题，暂时想不起来了，想起来再补上  

怕是自己64位系统的问题，还特意装了个xp虚拟机，结果貌似也跟这个没关系。。。  

具体步骤等我彻底成功了再贴出来  

PS：耗尽最后一丝耐心，终于根据下面这俩把vld这个扩展编译成功并且加载上了，http还是没搞定，回头再慢慢找原因了。  

这个可能相对靠谱：  
[http://1.lanz.sinaapp.com/?p=124  

  ](http://1.lanz.sinaapp.com/?p=124)这个有空再试验一下，看起来也稍靠谱：  
[](http://zgqwork.blog.51cto.com/1721633/495958)http://hyshucom.iteye.com/blog/1674498  http://zgqwork.blog.51cto.com/1721633/495958