+++
title= "改变extjs4.1中tooltip的整体颜色"
date= 2013-03-04T20:40:34+08:00
categories = ["js"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

1、尝试过用cls，style，都是在火狐，chrome，ie9上没问题，在ie8以下就有问题

2、尝试过bodyStyle，能行，但是你会发现有个令人讨厌的蓝边（原始样式）

然后在sencha论坛里发现有个哥们跟我遇到同样问题，是通过改baseCls解决的，确实好用，也不麻烦，贴出来大家共用，原帖地址（倒数第二个回帖）：http://www.sencha.com/forum/showthread.php?40155-Chang-background-color-in-a-tooltip

但我的做法比他多一步，就是我重写了Ext.tip.ToolTip因为我用了gray这个theme，所以前面那个anchor是个灰色的，和我整体颜色不搭：
```
Ext.define('Ech.ux.ToolTip', {  
extend: 'Ext.tip.ToolTip',  
alias: 'ech.tooltip',  
alternateClassName: 'Ech.ToolTip',  
baseCls:'gy-x-tip',  
anchorStyle:{  
borderColor:'#F1F7FD',  
borderLeftColor:'transparent',  
borderTopColor:'transparent',  
borderBottomColor:'transparent'  
},  
getTargetXY:function(){  
var me = this;  
var ret_parent = this.callParent(arguments);  
me.anchorEl.setStyle(me.anchorStyle);  
return ret_parent;  
}  
});
```
然后把所有带tip的css从ext-all-gray-debug.css里考出来，就改带background-color和border-color的就行了，值为transparent的不用管，我这里用的颜色是background-color: #F1F7FD;border-color: #BACDCE;