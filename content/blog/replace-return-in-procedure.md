+++
title= "在procedure中代替return的方法"
date= 2012-05-23T15:56:19+08:00
categories = ["mysql"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

```
DELIMITER //
DROP PROCEDURE IF EXISTS proc_test_leave;
CREATE PROCEDURE proc_test_leave()
begin
        declare ab varchar(10);
        label_leave:begin
        SET ab="1";
        if 1 then
                set ab="2";
                if 1 then
                        SET ab="3";
                        leave label_leave;
                end if;
        else
                leave label_leave;
        end if;
        end label_leave;
        select ab;
END;
//
DELIMITER ;

```