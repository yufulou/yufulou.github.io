+++
title= "mongodb本地表移库"
date= 2014-08-21T20:13:01+08:00
categories = ["mongo"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

在网上找这种本地移表的方法，一直没找着，自己写了个凑合用,有时候好不容易写几行基础数据不容易啊，尤其mongodb。。。每次还得带着字段名。。。回头万一设计时候出错了也比较好改~~
下面这种办法会报错：
```
db.runCommand({cloneCollection:"db1.table_to_move", from:"127.0.0.1:27017"});
{ "errmsg" : "can't cloneCollection from self", "ok" : 0 }

use db1;

var cursor = db.table_to_move.find();

use db2;

while(cursor.hasNext()){
db2.table_to_move.insert(cursor.next());
}
```