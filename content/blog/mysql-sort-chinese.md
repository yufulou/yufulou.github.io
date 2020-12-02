+++
title= "mysql utf8 中文字符 排序"
date= 2012-02-04T22:05:27+08:00
categories = ["mysql"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

最原始的帖子我打开不开，就贴这个地址吧：
http://adriano.blogbus.com/logs/14278134.html

跟我测试的结果一样，所以网上那么多order by convert(字段 using gbk)都是怎么搞出来的啊？




乘着放假，帮老大做个系统以减轻他的工作强度。

用什么做，当然是ruby on rails+mysql啦。

又是一番安装、配置。然后开始写代码。很多人遇到的中文问题，我在上一次做练习的时候已经遇到并解决了。

当时发现，当浏览器使用简体中文gb2312时，ruby代码中的中文没有问题，但是从数据库中的中文是乱码。换成utf8，则相反，mysql中的中文正常，ruby中的代码有问题。当时，相关配置都没有做特别调整。后来的解决方案很简单，将ruby代码用utf8的格式存放。我一直在用editplus来编辑，因此只要另存为utf8格式，则一切搞定(提醒一下，用rails以及script\generate自动生成时，默认都是ansi，如果涉及到中文一定要改为utf8)。

这一次驾轻就熟，直接用了上次的经验。因此中文显示一切正常。顶多在布局文件里加上charset设定。

然而一切写完以后，才发现，按照中文字段进行排序，其结果始终很奇怪。几番搜索核查之后，得知是因为utf8中中文字符的顺序是乱得，而非拼音排序！

解决的过程大致有四个阶段：

一、首先当然是google和baidu,下面是被转载最多的文章，基本上用"mysql 中文排序"搜索，前n页都是对此的转载。提供了2种方法

<font color="#ff6600" style="margin-top: 0px; margin-right: 0px; margin-bottom: 0px; margin-left: 0px; padding-top: 0px; padding-right: 0px; padding-bottom: 0px; padding-left: 0px; ">方法1，一种解决方法是对于包含中文的字段加上"binary"属性，使之作为二进制比较，例如将"name char(10)"改成"name char(10)binary"。</font>

<font color="#ff6600" style="margin-top: 0px; margin-right: 0px; margin-bottom: 0px; margin-left: 0px; padding-top: 0px; padding-right: 0px; padding-bottom: 0px; padding-left: 0px; "><strong style="margin-top: 0px; margin-right: 0px; margin-bottom: 0px; margin-left: 0px; padding-top: 0px; padding-right: 0px; padding-bottom: 0px; padding-left: 0px; ">&nbsp;方法2 ，</strong>如果你使用源码编译MySQL，可以编译MySQL时使用--with--charset=gbk 参数，这样MySQL就会直接支持中文查找和排序了。</font>

这里说的语焉不详，方法2，实验起来太过麻烦，而且我的实验机器是win，如何编译还不知道。而且实际上这意味着需要采用gbk作为中文存储的字符编码。方法1，较为简单，试验之，无效。原因后来才明白。

二、后来查到这个网页，使我有点绝望

http://www.5ivb.com/Info/68/3390242/

<font color="#ff6600" style="margin-top: 0px; margin-right: 0px; margin-bottom: 0px; margin-left: 0px; padding-top: 0px; padding-right: 0px; padding-bottom: 0px; padding-left: 0px; ">我研究了一下，初步了解：用二进制&nbsp;&nbsp;binary&nbsp;&nbsp;排序的前提是把&nbsp;&nbsp;gbk/gb2312&nbsp;&nbsp;的汉字编码以&nbsp;&nbsp;utf-8&nbsp;&nbsp;的格式存进去，所以一个汉字被保存成&nbsp;&nbsp;4&nbsp;&nbsp;个字节，而不是真正的&nbsp;&nbsp;3&nbsp;&nbsp;个字节。如果数据库中真的以&nbsp;&nbsp;utf-8&nbsp;&nbsp;的字符集保存，则二进制&nbsp;&nbsp;binary&nbsp;&nbsp;排序会得出错误结果（非拼音排序）。另外，以&nbsp;&nbsp;utf-8&nbsp;&nbsp;保存&nbsp;&nbsp;gbk/gb2312&nbsp;&nbsp;的编码会带来&nbsp;&nbsp;char_length()&nbsp;&nbsp;错误的问题。</font>

<font color="#ff6600" style="margin-top: 0px; margin-right: 0px; margin-bottom: 0px; margin-left: 0px; padding-top: 0px; padding-right: 0px; padding-bottom: 0px; padding-left: 0px; ">基本有结果了，看了&nbsp;&nbsp;mysql&nbsp;&nbsp;的文档后，明白了直接的排序办法没有！&nbsp;&nbsp;<br style="margin-top: 0px; margin-right: 0px; margin-bottom: 0px; margin-left: 0px; padding-top: 0px; padding-right: 0px; padding-bottom: 0px; padding-left: 0px; ">&nbsp;&nbsp;让我们期待&nbsp;&nbsp;utf8_chinese_ci&nbsp;&nbsp;的发布。&nbsp;&nbsp;<br style="margin-top: 0px; margin-right: 0px; margin-bottom: 0px; margin-left: 0px; padding-top: 0px; padding-right: 0px; padding-bottom: 0px; padding-left: 0px; ">&nbsp;&nbsp;在这之前只能考虑间接的办法了。</font>

三、于是开始决定放弃utf8，修改为gbk或者gb2312

查看了若干网页，试验若干次，终于发现一下几点：

1、我的浏览器没有gbk编码(可能没有安装)，它似乎看不了gbk的字符集的东西，那么想来很多浏览器默认也没有gbk编码，所以决定弃用gbk。(这一点没有仔细求证，或有不确之处，只是当时就这么想的)

2、要将我做好的系统整个改成gb2312，需要做以下的事情：

a.将所有的ruby文件改回ansi。

b.在mysql中，设置服务器、数据库、表以及要排序的中文字段的字符集编码均设为gb2312，将排序方式设为gb2312_chinese_ci ，其中，服务器的字符集和排序方式在my.cnf中设置，后面的三个可以使用sql命令alter database或者alter table来设置，当然我是用phpadmin设置的。

c.设置连接和客户端的字符集，其实我究竟设置的是哪一个我也不是很清楚，我设置的是ruby的database.yml

adapter: mysql  
database:   
username: root  
password:   
host:   
encoding: gb2312

d.上面已经将整个过程中可能配置的编码方式都修改为了gb2312，但是还有重要的最后一点，就是在网页中，输入中文的时候，必须当时网页是采用gb2312显示的，此时你输入进去的字符才是gb2312，否则在后来的显示就会出现乱码。

当然上述的过程中也许有些设置可以不需要，比如，也许只要设置字段的编码和排序方式即可。未加尝试。总之，过程过于复杂，因而我也放弃了这样的想法。当然许多刚刚开始做系统的人当然可以一试。

四、那么对于我的系统，我该怎么办呢？

1、认真地研读了mysql5的文档。看到了mysql的解决方案，那就是convert()函数和cast()函数，一一试来。

@test = Province.find_by_sql("select *, cast( prov_name as CHAR character set gb2312) collate gb2312_chinese_ci as ordername  from provinces order by ordername desc")

结果，不行，还是原来的奇怪样子。修改为使用convert函数

@test = Province.find_by_sql("select *, convert( prov_name using gb2312) as ordername  from provinces order by ordername desc")

还是一样。原因至今不明

2、最后，决定从ruby下手，查询的结果ruby会放在数组之中，每个数组元素是哈希表形式的一条记录。将查询的结果以新的方式重新排序不就可以了吗？

幸好我们有可爱的iconv。(注意有时候安装ruby时可能没有安装iconv，需要额外安装一下)。这是个负责在字符集间进行转换的函数库。如果要使用它，还需要在controller中加上 require 'iconv'

下面就是我的实现，当然也实验了多次，查看了半天ruby的文档才得到的结果

conv = Iconv.new("GBK", "utf-8")

ordername  from provinces order by ordername desc")  
@test1 = Province.find(:all).sort {|x, y| conv.iconv(x.prov_name) <=> conv.iconv(y.prov_name)}

它会按照省名的gbk字符进行排序，但是并不会修改其中的内容，也就是说显示时仍然是utf8格式。

终于搞定了，我也终于写完了，当然这过程我明白了许多其中的机制，字符集是复杂的问题。我会再写一篇来完成的写我的心得。下一篇吧。

这个第二个方法貌似靠谱的样子：

转自：http://www.024zq.com/?p=98

mysql 中文拼音排序

2011-10-09

客服那边需要我对一些酒店进行中文拼音排序，以前没有接触过，在php群里问了一些大牛。。得到了2种答案，都可以。哈哈·~  
以下既是msyql 例子  
表结构是utf-8的  
1.  

1.  SELECT `hotel_name`
2.  FROM `hotel_base`
3.  ORDER BY convert( `hotel_name`
4.  USING gbk )
5.  COLLATE gbk_chinese_ci

2.  

1.  SELECT `hotel_id` , `hotel_name` , ELT( INTERVAL( CONV( HEX( left( CONVERT( `hotel_name`
2.  USING gbk ) , 1 ) ) , 16, 10 ) , 0xB0A1, 0xB0C5, 0xB2C1, 0xB4EE, 0xB6EA, 0xB7A2, 0xB8C1, 0xB9FE, 0xBBF7, 0xBFA6, 0xC0AC, 0xC2E8, 0xC4C3, 0xC5B6, 0xC5BE, 0xC6DA, 0xC8BB, 0xC8F6, 0xCBFA, 0xCDDA, 0xCEF4, 0xD1B9, 0xD4D1 ) , ‘A’, ‘B’, ‘C’, ‘D’, ‘E’, ‘F’, ‘G’, ‘H’, ‘J’, ‘K’, ‘L’, ‘M’, ‘N’, ‘O’, ‘P’, ‘Q’, ‘R’, ‘S’, ‘T’, ‘W’, ‘X’, ‘Y’, ‘Z’ ) AS PY
3.  FROM hotel_base
4.  ORDER BY PY ASC

方法一较方法二简单些 呵呵 希望对迷惑的人有帮助~~~