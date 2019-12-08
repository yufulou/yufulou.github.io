+++
title= "增加俩php5.2.17的.gdbinit方法"
date= 2018-10-31T23:57:52+08:00
categories = ["php"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

最近，因为经常遇到print_cvs打印失败直接退出的问题，故做了两个方法，打印所有当前的变量名和单个变量，加到现有的.gdbinit就能用

同时记得老版本的print_zv打印的地址是错的，要把printf后面的参数改成[%p]

```text
define print_vars
    ____executor_globals
    set $p = $eg.current_execute_data.CVs
    set $c = $eg.current_execute_data.op_array.last_var
    set $v = $eg.current_execute_data.op_array.vars
    set $i = 0

    printf "Compiled variables count: %d\n", $c
    while $i < $c
        printf "%d = %s [%p] \n", $i, $v[$i].name, *$p[$i]
        set $i = $i + 1
    end
end

document print_vars
    print all var names
end

define print_var
    ____executor_globals
    set $p = $eg.current_execute_data.CVs
    set $c = $eg.current_execute_data.op_array.last_var
    set $v = $eg.current_execute_data.op_array.vars
    set $i = 0
    set $varname = $arg0
    set $found_var = 0

    printf "Compiled variables count: %d\n", $c
    printf "to print var name %s\n", $varname
    while $i < $c && 0 == $found_var
        if strcmp($varname, $v[$i].name) == 0
            set $found_var = 1
            if $p[$i] != 0
                printzv *$p[$i]
            end
        end
        set $i = $i + 1
    end
end

document print_var
    print single var, eg: print_var "var1"(without initial $)
end
```

同时，改写一个方法，原来的gdbinit逻辑是只要不是英文或数字，就打印8进制，现在先把原字符串打印出来：
```text
define ____print_str
    set $tmp = 0
    set $str = $arg0 
    printf "%s", $str
    printf "\""
    while $tmp < $arg1 
        if $str[$tmp] > 32 && $str[$tmp] < 127
            printf "%c", $str[$tmp]
        else 
            printf "\\%o", $str[$tmp]
        end 
        set $tmp = $tmp + 1
    end
    printf "\""
end
```