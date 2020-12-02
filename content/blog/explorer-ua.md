+++
title= "判断浏览器类型，系统类型的js和php代码"
date= 2014-07-29T11:39:33+08:00
categories = ["php", "js"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++


同一套判断方式的两种写法  

php：  

```
class BrowserDetector {
    public $UA = ""; //$HTTP_USER_AGENT的内容
    public $BROWSER_VERSION = array();
    
    /* 构造函数开始 */
    function __construct(){
        $this->UA = getenv(HTTP_USER_AGENT);
        $this->BROWSER_VERSION = array(
                "trident"=> false !== stripos($this->UA, 'Trident'), //IE内核
                "presto"=> false !== stripos($this->UA, 'Presto'), //opera内核
                "webKit"=> false !== stripos($this->UA, 'AppleWebKit'), //苹果、谷歌内核
                "gecko"=> false !== stripos($this->UA, 'Gecko') && false === stripos($this->UA, 'KHTML'), //火狐内核
                "mobile"=> 0 < preg_match('/AppleWebKit.*Mobile.*/i', $this->UA)
                            || 0 < preg_match('/AppleWebKit/i', $this->UA), //是否为移动终端
                "ios"=> 0 < preg_match('/(i[^;]+\;(U;)? CPU.+Mac OS X)/i', $this->UA), //ios终端
                "android"=> false !== stripos($this->UA, 'Android') || false !== stripos($this->UA, 'Linux'), //android终端或者uc浏览器
                "iPhone"=> false !== stripos($this->UA, 'iPhone') || false !== stripos($this->UA, 'Mac'), //是否为iPhone或者QQHD浏览器
                "iPad"=> false !== stripos($this->UA, 'iPad'), //是否iPad
                "microMessenger"=> 0 < preg_match('/MicroMessenger/i', $this->UA),
                "webApp"=> false === stripos($this->UA, 'Safari') //是否web应该程序，没有头部与底部
        );
    }
}
```

js版本：  

```
var browser={

        versions:function(){
            var u = navigator.userAgent, app = navigator.appVersion;

            return {
                trident: u.indexOf('Trident') > -1, //IE内核
                presto: u.indexOf('Presto') > -1, //opera内核
                webKit: u.indexOf('AppleWebKit') > -1, //苹果、谷歌内核
                gecko: u.indexOf('Gecko') > -1 && u.indexOf('KHTML') == -1, //火狐内核
                mobile: !!u.match(/AppleWebKit.*Mobile.*/)||!!u.match(/AppleWebKit/), //是否为移动终端
                ios: !!u.match(/(i[^;]+\;(U;)? CPU.+Mac OS X)/), //ios终端
                android: u.indexOf('Android') > -1 || u.indexOf('Linux') > -1, //android终端或者uc浏览器
                iPhone: u.indexOf('iPhone') > -1 || u.indexOf('Mac') > -1, //是否为iPhone或者QQHD浏览器
                iPad: u.indexOf('iPad') > -1, //是否iPad
                microMessenger: (null != u.match(/MicroMessenger/i)),
                webApp: u.indexOf('Safari') == -1 //是否web应该程序，没有头部与底部
            };
        }(),
        language:(navigator.browserLanguage || navigator.language).toLowerCase()
    }
```