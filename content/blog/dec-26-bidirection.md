+++
title= "26进制(A-Z)与10进制(1-10)的相互转换"
date= 2014-03-15T09:42:21+08:00
categories = ["php"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

```
    /**
     * 26进制（A-Z）转10进制int（1-10），服务于excel列数转换
     * @param string $colnum
     * @return number
     */
    public function transXlsColNumToInt($colnum){
        $result_int = 0;
        $column = trim($colnum);
        if(empty($colnum)) return $result_int;
        $column_len = strlen($colnum);
        $bit_curr = pow(26, ($column_len-1));
        $start_pos = 0;
        while($start_pos<=$column_len-1){
            $chr = substr($colnum, $start_pos);
            $result_int += (ord(strtoupper($chr))-64) * $bit_curr;
            $bit_curr /= 26;
            $start_pos++;
        }
        return $result_int;
    }
    
    /**
     * 10进制int（1-10）转26进制A-Z，服务于excel列数转换
     * @param int $intnum
     * @return string
     */
    public function transIntToXlsColNum($intnum){
        $result_num = "";
        if(empty($intnum) || !is_numeric($intnum)) return $result_num;
        $remainder = 0;
        $int_last = doubleval($intnum);
        while($int_last>0){
            $remainder = fmod($int_last-1, 26) + 1; // 太大的数，使用%会导致溢出；由于26进制，应该是0-25，所以这里要减掉当前位置上多出的1
            $int_last = doubleval(floor(($int_last-1) / 26));
            $result_num = chr($remainder+64) . $result_num;
        }
        return $result_num;
    }
```