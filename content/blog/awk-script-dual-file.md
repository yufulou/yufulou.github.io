+++
title= "从同事那边要来的一个两个文件去重的awk脚本"
date= 2015-12-16T15:30:34+08:00
categories = ["linux"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

从同事那边要来的一个两个文件去重的awk脚本，很实用，记录一下，不是比较整行，故用的时候得看看
```
#!/bin/bash

createSecondFile(){
awk -F"\t" '
ARGIND == 1 { a[$3] = 1; next;}
ARGIND == 2 {
if (a[$2] == 1) {
next;
}
printf("%s\t%s\n",$1, $2);
}
' $1 $2 > $3
}

count=$1

for i in `seq 0 ${count}`;do
sqlfile=sql_${i}.tmp;
updatefile=update_${i};
outfile=sql_${i}.second.tmp;
echo "Generate ${outfile}"
createSecondFile $updatefile $sqlfile $outfile;
done
```