---
title: 算法@MD5算法逆向与还原
date: 2022-02-11 14:47:37
tags:
---
# MD5算法逆向与还原

## 算法的识别

md5算法很好识别，一般会先调用MD5Init，然后调用MD5Update，最后调用MD5Final，这里多次调用md5update，实际上完成的是字符串的加盐

如果不是很确定，可以先google搜索MD5Init那里的常量，一般可以搜索到算法

![Untitled](https://s2.loli.net/2022/02/11/P7VRm2bgYkOeiQH.png)

## MD5算法还原

google搜md5.c，找到MD5的实现，编写算法还原工具，需要注意MD5Init的常量需要逆向获得

## 混淆的MD5算法还原

对于混淆的算法，直接硬看一般什么也看不出来，不过可以通过参数引用来进行逆向分析

这个jni函数只有一个参数，类型是一个字符串，源字符串通过这个参数传递进来，我们把它重命名为input

![Untitled](https://s2.loli.net/2022/02/11/SoYUZGWs7dckjLX.png)

查看它的交叉引用

![Untitled](https://s2.loli.net/2022/02/11/PUIgjzAxS4pJGyt.png)

查看第一处input引用，通过v15传参，接着可以分析v15的引用情况，同时发现v15返回值是s，返回值也要关注，这里sub_F794传递了v15和s，所以可以通过frida hook并用hexdump观察内存中的字节码进行分析

![Untitled](https://s2.loli.net/2022/02/11/ubFiv6GrXlHpRYc.png)

最终定位到关键函数，再找到MD5Update的位置，hook参数，查看这些参数字符串，将这些参数字符串拼接代入到md5算法中如果结果一致就可以编写还原工具了

![Untitled](https://s2.loli.net/2022/02/11/ANbr6P4xtyG3Lip.png)

## 深度修改MD5算法还原

有些MD5算法即使hook到MD5Update传递的加盐字符串，算出的结果仍然不一致，这种情况就很有可能是动了md5的transform

![Untitled](https://s2.loli.net/2022/02/11/HIPoiG5WbJNsBAp.png)

transform函数里面非常复杂，也有非常多的常量，可以在google搜索到包含这些常量的源文件，然后找到这些常量

![Untitled](https://s2.loli.net/2022/02/11/eomnxTwAyXf9Cig.png)

接着通过动态trace指令，把目标函数所有指令trace下来，里面必然包含了这些常量，最后写脚本比对不存在的常量，那么不存在的常量就是被修改的，在trace文件中找到被修改的常量，然后patch md5算法，就可以实现算法还原工具了 

![Untitled](https://s2.loli.net/2022/02/11/oYV9NsnSEkh5rLd.png)

这里缺少的常量是下面两个

![Untitled](https://s2.loli.net/2022/02/11/aAdrLZD9qkcUO1f.png)

缺少的常量在代码中的位置 

![Untitled](https://s2.loli.net/2022/02/11/YxT4iBVlfaqs8Hk.png)

可以先找到在它前面的常量0xfd469501，+CCC地址是处理这个常量的地址，执行到这里时x10里面保存的值是0xfd469501

![Untitled](https://s2.loli.net/2022/02/11/uHqs8X3d4CZ7Umi.png)

在ida中定位到位置

![Untitled](https://s2.loli.net/2022/02/11/hrOW1zMovw2gibe.png)

![Untitled](https://s2.loli.net/2022/02/11/3lufRvB2jkOp8tG.png)

4249261313转换为16进制就是常量0xfd469501

![Untitled](https://s2.loli.net/2022/02/11/VpAR3uHM5S2Pw6f.png)

那么它的下一个常量可能就是修改过的常量了

![Untitled](https://s2.loli.net/2022/02/11/k9QYq8ZmBjhsGlp.png)

但是在trace文件中找不到这个值，也就是说上图中的值应该不是

按tab找到在汇编中的位置，因为源码中对这些常量用的是加法，所以这里看到ADD就可以在trace文件中搜索c428

![Untitled](https://s2.loli.net/2022/02/11/9Rfc6D53oJuadhm.png)

在trace文件中找到了这个位置，根据之前常量所在位置可以确定这个值是0x699880d8，缺失的值修改为它就可以了

![Untitled](https://s2.loli.net/2022/02/11/4DUCcVWq8tp56Gw.png)

继续找另一个缺失的常量0xc33707d6，同样的先定位它前面的常量0x21e1cde6

![Untitled](https://s2.loli.net/2022/02/11/kTw7xQAN95s4Cjg.png)

在0x21e1cde6和0xf4d50d87之间找了所有add附近的常量都不太像，最后在0x21e1cde6前面的位置找到了，它的值是0xc30737d6，缺失的另一值也找到了，可见在指令执行时位置不绝对按照源码中的顺序来执行的

![Untitled](https://s2.loli.net/2022/02/11/IZj8S95HPUeuiLY.png)

最后把这两个改变的常量代入到md5算法工具代码中使用，计算出的结果一致了