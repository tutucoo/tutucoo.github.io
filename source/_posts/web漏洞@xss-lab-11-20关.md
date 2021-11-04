---
title: web漏洞@xss-lab 11-20关
date: 2021-04-19 10:05:55
cover: https://i.loli.net/2021/04/19/jJxAgq9OBnpksad.jpg
---

# xss-lab 11-20关

## 第十一关 Referer存在的xss

套路还是跟第十题类似，按钮被隐藏  输入exp进行测试 

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled.png](https://i.loli.net/2021/04/19/JDeZU7HwpnBFVWR.png)

无论是单引号还是双引号都无法对其进行封闭，猜测引号被过滤了

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%201.png](https://i.loli.net/2021/04/19/DrF9S1YyIGm7lQ4.png)

我们看下源码，到底进行了哪些防护

可以看到，不仅对t_sort的字符串进行了实体化操作，而且增加了对HTTP_REFERER的过滤，不允许字符串中包含大小括号

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%202.png](https://i.loli.net/2021/04/19/5ZoxEs1yFjLmHzk.png)

这样一来，由于无法用单引或双引号进行闭合，就只能用大小括号进行闭合，现在连大小括号都过滤了，因此无法闭合。

山穷水覆疑无路，柳暗花明又一村，这里接收了HTTP_REFERER参数，我们就可以对这个参数进行输入

隐藏的输入框有个t_ref，通过源码比对也知道，它就是用来接收Referer的值的，但是这里直接对其进行请求是无法注入的，不过可以通过hackbar进行注入

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%203.png](https://i.loli.net/2021/04/19/wGI7cZ8pbktgrhq.png)

最终exp

```
 " onclick=alert() type="button"
```

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%204.png](https://i.loli.net/2021/04/19/nSeVfwj48ogsMOu.png)

## 第十二关 User-Agent存在的xss

看一下隐藏的元素，有个ua，这跟上一关应该类似，只不过这次是利用的User-Agent

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%205.png](https://i.loli.net/2021/04/19/yPn3jLuleRoGIUW.png)

就不多说了，最终exp

```
 " onclick=alert() type="button"
```

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%206.png](https://i.loli.net/2021/04/19/b92h3QWoyuFR5ca.png)

## 第十三关 Cookie存在的xss

t_sort可注入，但是跟之前的一样，t_sort无法用引号闭合，而且过滤了括号，因为没法闭合，所以无法执行我们的代码，看到有个t_cook，就明白了，这是利用了cookie

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%207.png](https://i.loli.net/2021/04/19/x1yQoTfUE5ZDqN9.png)

跟之前两关一样的操作，不过并没有xss

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%208.png](https://i.loli.net/2021/04/19/HPiTjJQOM1R65Cv.png)

看了一下源码，确实是利用的cookie

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%209.png](https://i.loli.net/2021/04/19/QmPyVUtW3FZhiMT.png)

不过这里需要添加user作为key

```
 user=" onclick=alert() type="button"
```

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2010.png](https://i.loli.net/2021/04/19/mnzp2BJiLedWlPE.png)

## 第十四关 exif xss

这是一个罕见的exif xss，但是由于exifviewer好像不支持上传了，现在也已经无法复现了

## 第十五关 angular js xss

这一关通过第十四关跳转过来，有个src的参数

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2011.png](https://i.loli.net/2021/04/19/AhvwzatqDFE64nB.png)

发现src的值会注入到这里，ngInclude是angular js独有的，相当于php的include函数 

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2012.png](https://i.loli.net/2021/04/19/cz5pqknslPECwbo.png)

看下源码，是通过echo执行了这里的代码

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2013.png](https://i.loli.net/2021/04/19/NAQEn4Ykwop8Pia.png)

我们包含一下level1进来，使用下面的exp，level1的xss就可以直接拿来用了

```
 http://192.168.0.21/xss-lab/level15.php?src='level1.php?name=<script>alert(1)</script>'
```

执行没有效果，考虑是有过滤，看下源码，发现调用了htmlspecialchars函数，对括号、引号、&号进行了过滤，但是这里奇怪的是，使用<img又是可以的，htmlspecialchars函数是会过滤括号的，这里使用了括号怎么就可以了？

```
 http://192.168.0.21/xss-lab/level15.php?src='level1.php?name=<img src=1 onerror=alert()>'
```

## 第十六关 空格过滤

进来看来keyword参数，随便试一试

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2014.png](https://i.loli.net/2021/04/19/AVTQzdoZvcHSRwP.png)

可以看到对一些字符进行了过滤，替换成了空格的html实体编码&nbsp

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2015.png](https://i.loli.net/2021/04/19/8X7MUOld4NvCi1k.png)

看下过滤的方式，可以看到对空格和/进行了过滤

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2016.png](https://i.loli.net/2021/04/19/xjazDSlE6FmWrcT.png)

空格可以用%0D进行代替，%0D是URL编码表示归位

最终exp

```
 http://192.168.0.21/xss-lab/level16.php?keyword=%3Cimg%0Dsrc=1%0Donerror=alert()%3E
```

## 第十七关 flash控件xss

由于现在对flash不再支持，所以插件无法显示，但是不影响复现

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2017.png](https://i.loli.net/2021/04/19/TApZnzVWtFGIb4h.png)

对两个参数进行操作，发现arg01作为变量，而arg02作为值，这样就可以用arg02测试xss了

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2018.png](https://i.loli.net/2021/04/19/SHbVlQJAK3U1vth.png)

可以看到对两个参数都进行了引号、括号、&号过滤

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2019.png](https://i.loli.net/2021/04/19/6O9NsIvS3RcjloL.png)

经过测试，括号用``代替，绕过htmlspecialchars函数，因为是embed控件，查看它的属性，发现只有type没有用，但是这个属性并没有能够触发xss，查看它支持的事件，采用onmouseover事件进行测试(onmouseover前面必须有空格，属性之间必须有空格)，最终exp如下，

```
 http://192.168.0.21/xss-lab/level17.php?arg01=dddd&arg02= onmouseover=alert`1`
```

## 第十八关 flash控件xss

首页也是类似的

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2020.png](https://i.loli.net/2021/04/19/yx3IUgwDMZfmPoF.png)

exp同第十七关完全一样

```
 http://192.168.0.21/xss-lab/level17.php?arg01=dddd&arg02= onmouseover=alert`1`
```

## 第十九关 flash控件xss

本关进行之前需要了解flash xss:

Flash XSS攻击总结 杀死那个石家庄人/ 菲哥哥:https://www.secpulse.com/archives/44299.html

这一类的xss比较少见，更多技术细节参见：

那些年我们一起学xss:https://wizardforcel.gitbooks.io/xss-naxienian/content/14.html

payload flash xss

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2021.png](https://i.loli.net/2021/04/19/4manwFxdISEet1M.png)

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2022.png](https://i.loli.net/2021/04/19/VNjG5KPsmaHAyDc.png)

## 第二十关 flash控件xss

这一题用到了zeroclipboard xss，具体可以参考这篇文章：

https://www.freebuf.com/sectool/108568.html

国内广泛使用了zeroclipboard.swf，主要的功能是复制内容到剪切板，中间由flash进行中转保证兼容主流浏览器，具体做法就是使这个透明的flash漂浮在复制按钮之上

看一下源码：

使用jpexs反编译(swf反编译工具：https://github.com/jindrapetrik/jpexs-decompiler

)

原因显而易见，Externalinterface.call第二个参数传回来的id没有正确过滤导致xss

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2023.png](https://i.loli.net/2021/04/19/uTsEYGUmQexPgXL.png)

payload

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2024.png](https://i.loli.net/2021/04/19/GrR5nM8Jvp1zA4x.png)

![xss-lab%2011-20%E5%85%B3%200c31978e6d1a400cb1c8251f5fab4f1a/Untitled%2025.png](https://i.loli.net/2021/04/19/dkca5GHt8TpsRZf.png)