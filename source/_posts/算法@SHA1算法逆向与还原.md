---
title: 算法@SHA1算法逆向与还原
date: 2022-02-11 14:51:37
tags:
---
# SHA1算法逆向与还原

## 概述

sha1算法加密后的字符串长度是40位，sha256长度是64位

## 一般sha1算法还原

目标程序加密结果跟SHA1算法对不上

![Untitled](https://s2.loli.net/2022/02/11/Lg8UjlFRhq2X4GE.png)

![Untitled](https://s2.loli.net/2022/02/11/mMAb8kfg1hQFePI.png)

看下伪代码，SHA1算法跟MD5是类似的，也会用调用SHA1Update函数  

![Untitled](https://s2.loli.net/2022/02/11/GOk1WbZ6itMNxhm.png)

把盐字符串拼接上就对了

![Untitled](https://s2.loli.net/2022/02/11/MlmW4afo79H2Jp3.png)

## SHA1Init常量变形还原

运行目标程序，发现加密跟标准sha1对不上

![Untitled](https://s2.loli.net/2022/02/11/7SdH6eTbUVpD2Er.png)

![Untitled](https://s2.loli.net/2022/02/11/PBh4l57VRfKEISz.png)

看下伪代码，发现SHA1Init函数中的常量跟标准SHA1算法不一样了，把标准SHA1算法代码中的常量替换成目标程序中的SHA1常量进行还原

![Untitled](https://s2.loli.net/2022/02/11/usIrEDJkvlQHoCp.png)

![Untitled](https://s2.loli.net/2022/02/11/OrZhUpVjyRD2lGw.png)

把开源的sha1源码拷贝到Clion，把常量修改一下进行调用就可以实现算法还原工具了

![Untitled](https://s2.loli.net/2022/02/11/beXuR6lZ95afTY3.png)

![Untitled](https://s2.loli.net/2022/02/11/ZnMyHhiwK3AFatk.png)

## SHA1Transform变形还原

逆向SHA1Init里的常量保存一致，SHA1Update的盐字符串也一致，结果却不一样，可以考虑是SHA1Transform函数的常量值被替换了

SHA1Transform是SHA1Update函数内部的一个函数，里面有很多常量参与运算，找出所有的常量再在标准算法里比对（也可以通过脚本进行筛查）

SHA1算法里的k1-k4就是transform里面的几个常量

![Untitled](https://s2.loli.net/2022/02/11/fxPoqDUZE8dSls6.png)

如果有不一致的地方修改回来，再编写算法还原工具

## SHA1Update字符串OLLVM混淆

可以看到这里字符串变成一个变量了

![Untitled](https://s2.loli.net/2022/02/11/t5gJAh38wxTWC9N.png)

查看这几个变量的交叉引用会跟踪到非常复杂的混淆

![Untitled](https://s2.loli.net/2022/02/11/GxM2ZbIzWTUlK5X.png)

对于这种情况可以通过frida hook这个变量地址直接打印出字符串再代入到算法工具中进行算法还原

![Untitled](SHA1%E7%AE%97%E6%B3%95%E9%80%86%E5%90%91%E4%B8%8E%E8%BF%98%E5%8E%9F%20318b528f70494ee78f6f5c4b63619467/Untitled%2013.png)