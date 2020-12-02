+++
title= "关于循环fread后push数组，造成内存异常大并越界问题的思考"
date= 2013-02-01T20:58:36+08:00
categories = ["php"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

**原问题：**

php版本5.3.6，线程安全版本，debian平台，php的memory_limit是16M
event.txt文件只要有行就行，每行最好长一点，比较好重现，比如：
11111111111111111111111111111111111111111111
22222222222222222222222222222222222222222222
脚本：
```
$fp = fopen("event.txt", "r");
if(!$fp)exit(1);
$arr = array();
while(true){
$line = null;
$event = null;
flush();
gc_collect_cycles();
var_dump(number_format(memory_get_usage(true)));
$line = fgets($fp, 4096 * 1024);
var_dump(number_format(memory_get_usage()));
array_push($arr, $line);
if(feof($fp))rewind($fp);
}
```
通过后台命令行执行这个，会发现，memory_get_usage(true)会隔一段时间上升4M左右（没算，目测可能正好就是4M），而memory_get_usage()一直不大，一会就Allowed memory size of 16777216 bytes exhausted了
最终显示：
。。。。
。。。。
。。。。
string(10) "13,893,632"
string(9) "1,968,844"
string(10) "13,893,632"
string(9) "1,968,988"
string(10) "13,893,632"
string(9) "1,969,132"
string(10) "13,893,632"
string(9) "1,969,276"
string(10) "13,893,632"
string(9) "1,969,364"
string(10) "13,893,632"
string(9) "1,969,436"
string(10) "13,893,632"
string(9) "1,969,580"
string(10) "13,893,632"
string(9) "1,969,724"
string(10) "13,893,632"
Fatal error: Allowed memory size of 16777216 bytes exhausted (tried to allocate 4194305 bytes) in /home/eChance/eChance/net_manager.02.02.static/web/debug/test_loop_read_file.php on line 11
可见，最后一个fgets搞出了内存不够
问题如下：
文档说：memory_get_usage(true)显示的应该是实际分配的内存，memory_get_usage()应该是emalloc的内存，那他们俩的差值是怎么产生的内存，为什么没有释放？该怎么释放？emalloc的数据在这个例子中，分配的是$arr变量的内存吧？
memory_get_usage(true)也才13M多，可能是因为fgets后会再增加4M导致内存溢出，那就证明溢出的是fgets使用的一个内存区域，可我不理解他为什么一直增加不释放？
如果memory_get_usage(true)显示的是包括分配的缓存部分，memory_get_usage()是实际变量的话，那memory_get_usage()还远远不到memory_get_usage(true)的大小，也不应该重新申请啊

**想到的：**

1、假设memory_get_usage(true)-memory_get_usage()主要就是数组的管理内存。
2、每次memory_get_usage(true)增加和fgets请求的大小一致，或许是因为fgets确实需要分配内存，但使用后不释放，当做下一次fgets的缓存和数组的管理内存，直接使用，或许是避免每次都重新分配内存的一种优化方式吧。当这块内存被占满后，再进行下一次分配，所以之前的很多次提示memory_get_usage(true)已经是13.25M，但fgets时却没有提示，由于他也依然在用之前分配而未释放的这段内存区域。
3、即使$arr=null也不减小或许也是同理，虽然不减小，但也不会像之前一样继续增加内存分配，目的也是避免多次重新分配内存。
4、由于管理内存和fgets缓存都在用每次预分配的4M，可能会造成缓存和数组管理内存之间产生一定的内存碎片，间接造成memory_get_usage(true)过分的大。对于这种每次都彻底读出一个小文件全部内容的程序，应该用file函数可以得到很大优化

**验证：**

代码如下：
```
$arr = array();
while(true){
$events = null;
flush();
gc_collect_cycles();
var_dump(number_format(memory_get_usage(true)));
$events = file("event.txt");
var_dump(number_format(memory_get_usage()));
$arr = array_merge($arr, $events);
}
```
执行效果如下：
。。。。
。。。。
。。。。
string(9) "4,528,496"
string(9) "5,767,168"
string(9) "4,530,356"
string(9) "5,767,168"
string(9) "4,532,088"
string(9) "5,767,168"
string(9) "4,533,880"
string(9) "5,767,168"
string(9) "4,535,712"
string(9) "5,767,168"
string(9) "4,537,492"
^C
最终我ctrl+c停止了，可以看到，memory_get_usage(true)与memory_get_usage()相差很小，确实有很大的优化效果，也验证了，管理内存绝对不会需要10倍多那么离谱。。。之前的memory_get_usage(true)超大很有可能是内存碎片造成。
暂时就当个想法吧，以后遇到这种问题先这么解决了~~回头有时间研究一下内存管理器的源码验证一下~~这个结论确实是瞎猜的算是，希望各位有啥新的想法随时来此赐教~~谢谢~~