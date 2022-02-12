---
title: trace技术@trace非标准算法
date: 2022-02-12 11:03:37
tags:
---
# trace非标准算法

如果目标sign加密采用了非标准算法，通过采用trace的方式进行算法还原

动态调试目标，利用IDA脚本可以对目标的so文件进行trace

trace过程中会碰到很多系统库的的函数，脚本对地址进行了判断，如果不是在目标so文件中的地址则不进行trace

![image-20220212110435364](https://s2.loli.net/2022/02/12/b14JWzcaYry2gt7.png)

这里指定一下so的名称以及指定函数的起始地址和结束地址

![Untitled](https://s2.loli.net/2022/02/12/jExiZ8uA2XTJINM.png)

IDA中进行动态调试，在算法函数断下后，执行trace脚本，如果此时程序有多个线程，要把其他线程挂起，防止反调试，之后要再执行resume_process()恢复进程

![Untitled](https://s2.loli.net/2022/02/12/7RpBxdYLoa8jE5M.png)

用IDA进行trace过程中，会产生trace日志，使用命令tail -f xxx.log可以动态看trace的日志有没有增加

![Untitled](https://s2.loli.net/2022/02/12/4lmzfbJpEYuswCt.png)

trace有两种方法：

1、直接在trace日志里面搜索加密后的数据，然后向前推算出算法

2、从加密函数开始处逆向出算法

先使用第2种方法，trace完之后找到函数开始的位置

![Untitled](https://s2.loli.net/2022/02/12/23deRlYWAjgipkL.png)

向上搜索找到两个参数，第一个参数是随机值，第二个参数是长度

![Untitled](https://s2.loli.net/2022/02/12/Mk9eOCFG7rXR5tl.png)

搜x0的值，发现x9这里使用了

![Untitled](https://s2.loli.net/2022/02/12/aLvF1z7p3BZuoWb.png)

再继续找x9，发现这里进行load，0x66其实就是f，也就是随机字符串的第一个字符

![Untitled](https://s2.loli.net/2022/02/12/soN9TI5ubztamxp.png)

因为这个地址+5f0的位置保存了随机字符串，那么trace时这个地址应该会保存字符串中的所有字符，全局搜一下，发现果然是随机字符串

![Untitled](https://s2.loli.net/2022/02/12/nf69kOWqsrecQF3.png)

再看第二个字符，000000725eb3f1b0是第一个字符，x9的值是000000725eb3f1b1也就是第2个字符

![Untitled](https://s2.loli.net/2022/02/12/QEkNzD8y4TgtR6n.png)

再往上看x9是哪里来的，可以看到是x0+1得来的，x9是1

![Untitled](https://s2.loli.net/2022/02/12/vScQUGqM8WyZto6.png)

继续搜这个地址

![Untitled](https://s2.loli.net/2022/02/12/fxNbJol3IjW6qky.png)

这里b7过后直接是b9，也就是说字符串下标为8的位置有特殊处理 

![Untitled](https://s2.loli.net/2022/02/12/vETx2tISjMfwyuz.png)

对比下输入和输出，可以看到第9位是一个横杠符号

![Untitled](https://s2.loli.net/2022/02/12/odmM6yRE52Ih1un.png)

搜一下b8

![Untitled](https://s2.loli.net/2022/02/12/yg258n6OtDwqobr.png)

b8被加载到x9，然后w23存到x9，x23的值是2d，也就是“-”

![Untitled](https://s2.loli.net/2022/02/12/mXeDPEgzMU92FBR.png)

下标为8、下标为d的位置都是直接被-替换，但是到了fUC2就不一样了，替换成了4UC2

![Untitled](https://s2.loli.net/2022/02/12/JzwE7XWLk1PpKog.png)

先把前两个-替换掉，再分析后面的算法

![Untitled](https://s2.loli.net/2022/02/12/f3hLQJSlGXKTYOC.png)

4UC2的4位置下标为e，通过下图中的地址，可以看到分别对每个下标字符进行了处理，而e并不存在

![Untitled](https://s2.loli.net/2022/02/12/ViBZWsagd74oEPN.png)

单独搜一下e

![Untitled](https://s2.loli.net/2022/02/12/S81ntr9RubqTAiL.png)

这里把w23存到x9位置，w23的值就是4

![Untitled](https://s2.loli.net/2022/02/12/wNgL3UZzI9yocK4.png)

看一下多次的结果，这个位置每一个都是4，说明是写死的 

![Untitled](https://s2.loli.net/2022/02/12/X2qxnO79AGF8ET5.png)

所以把这个位置填充为4

![Untitled](https://s2.loli.net/2022/02/12/Am5oeDhVKpTPCrn.png)

然后再看后面的，同样也是被替换成了-号，它的位置是2

![Untitled](https://s2.loli.net/2022/02/12/cHFjESWs5dLGrRT.png)

所以单独搜一下它的地址，发现确实被替换成了-号 

![Untitled](https://s2.loli.net/2022/02/12/hNFBI4unPyarHY6.png)

![Untitled](https://s2.loli.net/2022/02/12/IVZOXrBymfapGPv.png)

所以这个位置也赋值为-

![Untitled](https://s2.loli.net/2022/02/12/QTXUy6nrE5Ajboi.png)

接下来到了7这个位置又不一样了，变成了Seq

![Untitled](https://s2.loli.net/2022/02/12/319ZmNfIlixq8JA.png)

还是搜字符所在的地址，看下c7的位置的值是怎么来的 

![Untitled](https://s2.loli.net/2022/02/12/4QRKsxmYLl2paUn.png)

单独搜一下这个值

![Untitled](https://s2.loli.net/2022/02/12/zikgx6Pw2sjq39v.png)

这个值是从c7拿出来的，它保存的值是0x71，也就是q

![Untitled](https://s2.loli.net/2022/02/12/VAfPjpGckvhSX1s.png)

看一下这个值0x71是哪里来的，这个地址的引用有好几处，看下293行

![Untitled](https://s2.loli.net/2022/02/12/JltcoAfGIHyiaOK.png)

可以看到值来自[sp,#0xc]

![Untitled](https://s2.loli.net/2022/02/12/48R12iQ3kAK9dNe.png)

往上找[sp,#0xc]

![Untitled](https://s2.loli.net/2022/02/12/h1a3MgXpvSs25Bq.png)

可以看到w28的值存到[sp,#0xc]中了，w28的值来自于[x0,#0x18]，

![Untitled](https://s2.loli.net/2022/02/12/rlGBTwoOVhRQb6I.png)

x0+0x18实际上就是输入字符串的下标

![Untitled](https://s2.loli.net/2022/02/12/Zv51S7ieCdFnM8p.png)

这个算法就是这样，输入字符串下标18的位置给输出字符串17的位置，原位置由-代替

![Untitled](https://s2.loli.net/2022/02/12/sdpCcSluGzYRDvJ.png)

然后再看最后两位不一样的

![Untitled](https://s2.loli.net/2022/02/12/743F8HTlUgRprzP.png)

最后的2个地址看一下它的值

![Untitled](https://s2.loli.net/2022/02/12/FP9dgJlVuCZ3rMn.png)

d0的值是31，d1的值是75，对应1u，所以现在要找最后两位

![Untitled](https://s2.loli.net/2022/02/12/pkeh6mfnxZqKMuP.png)

搜索d2，但是没有结果，说明它并没有读取输入字符串这个位置的值 

![Untitled](https://s2.loli.net/2022/02/12/BeKVv12iJpHzUDT.png)

看下输入的引用，可以看到最后2个位置它直接写进去了 

![Untitled](https://s2.loli.net/2022/02/12/OtubxYCGB6PmoK8.png)

可以看到是赋值给了输入字符串

之前分析过输入字符串的地址赋值给了x0

![Untitled](https://s2.loli.net/2022/02/12/ywA5dJtO1KUIWHV.png)

要想找到最后两位，一定是对x0的某个偏移进行赋值，搜过x0=搜不到结果，可以搜索[x0,

下图对0x23偏移进行赋值，也就是下标35的位置赋值为0x34，也就是4

![Untitled](https://s2.loli.net/2022/02/12/mG6IknpMfPNdT5H.png)

下标34的位置也找到了，赋值为0x62，也就是b

![Untitled](https://s2.loli.net/2022/02/12/4Zk1wibxRLNOaBG.png)

赋值的位置找到了，还要找到这两个值的来源

先看最后一个，它的值来自地址x8，x8的值是c28c

![Untitled](https://s2.loli.net/2022/02/12/1KlWNqGETjoihm4.png)

因为x8是一个指针所以判断它是一个数组

![Untitled](https://s2.loli.net/2022/02/12/9qHXNwJELSoIFPn.png)

看下这个地址被赋值的情况，x9+x23的结果就是c28c，x23的值是0x4

![Untitled](https://s2.loli.net/2022/02/12/Z84xwekqH25soyO.png)

继续看x9的来源，来自c000

![Untitled](https://s2.loli.net/2022/02/12/rxsIZPmp7BGtF3e.png)

搜c000，但是只有这个结果

![image-20220212111120937](https://s2.loli.net/2022/02/12/iMXfy4FchstudZN.png)

这里卡住了，就直接搜0x34

这里搜索的时候可以用正则表达式

![Untitled](https://s2.loli.net/2022/02/12/76V1ILPBXHgRhmw.png)

0x34前面加4个点，这样包含0x34的也可以搜到，但是还是搜不到什么

![image-20220212111153857](https://s2.loli.net/2022/02/12/Pi9EMtaW5s1LDHb.png)

静态看下这里 

![Untitled](https://s2.loli.net/2022/02/12/UeAPcY9fJaoIq3p.png)

看下v14的引用

![Untitled](https://s2.loli.net/2022/02/12/2IpacPR4Z9NWgjz.png)

v14从一个字符串中拿一个字符作为值 

![Untitled](https://s2.loli.net/2022/02/12/EMOTFoj254vDkRG.png)

找到c70地址引用了它 

![Untitled](https://s2.loli.net/2022/02/12/XowB2Njn5ZcKq6T.png)

然后在trace日志里找到了对应的位置，所以0x71cc20c000就是字符串的地址 

![Untitled](https://s2.loli.net/2022/02/12/yUgS8eTFl2xk4tR.png)

然后0xc000+0x288，最后+0x4，在字符串对应数字4

![Untitled](https://s2.loli.net/2022/02/12/uVMzx2N45rYnJbe.png)

0x4来自[sp,#0x18]

![Untitled](https://s2.loli.net/2022/02/12/HsC8DpzngNWASuP.png)

它的值从w23传进来的

![Untitled](https://s2.loli.net/2022/02/12/mo34iREeqrUVStZ.png)

w23是通过w23&0xf得到的,x23的值是0x94，也就是0x94&0xf

w23是从[sp,#0x64]来的

![Untitled](https://s2.loli.net/2022/02/12/BT8wJ614zMhupjq.png)

搜索到上一次出现[sp,#0x64]的地方，数据来自w1，而w1的值就是0x94

![Untitled](https://s2.loli.net/2022/02/12/j2scl49ztV81HmQ.png)

w1又来自于w4，w4是w4跟w9异或的结果 

![Untitled](https://s2.loli.net/2022/02/12/47MjJnDokhLvTrl.png)

w4的值是0x75，w9的值是0xe1，0x75的值来自于输入字符串

![Untitled](https://s2.loli.net/2022/02/12/P2ECYHrvo8MAs7V.png)

搜一下这行，发现这个地址都是异或

![Untitled](https://s2.loli.net/2022/02/12/v3hfOXks4aEbByP.png)

看到了e1的值，是由D0跟31进行异或，31也是来自于源字符串

![Untitled](https://s2.loli.net/2022/02/12/nCeBslKFiP4q8EN.png)

d0的值来自6d和bd的异或，6d也是来自源字符串

![Untitled](https://s2.loli.net/2022/02/12/DScOKsbEYTmJ653.png)

这就发现了一个规律，异或的其中一个值都来自源字符串，而另一个值是上一个值异或的结果

![Untitled](https://s2.loli.net/2022/02/12/5TbqKrhSRF4QUsV.png)

对比一下源字符串，确实一致

![Untitled](https://s2.loli.net/2022/02/12/J2PIn5hYERSAB3f.png)

![Untitled](https://s2.loli.net/2022/02/12/64iLzKt8WyM1fuw.png)

写算法测试一下，在循环中进行异或

![Untitled](https://s2.loli.net/2022/02/12/KbFVfJwv25e7Plq.png)

这里的值还不一样，初始值其实是0xff

![Untitled](https://s2.loli.net/2022/02/12/SOGTMr9u4d6b8ya.png)

这个结果就正确了

![Untitled](https://s2.loli.net/2022/02/12/RkncgCFrBKJVM4I.png)

但是到了d5后面结果就不对了

![Untitled](https://s2.loli.net/2022/02/12/nkrHPecSAbjJXim.png)

打印一下下标，下标18的位置就不一样了，说明可能是下标17的位置有问题，因为下标17进行了单独的处理

![Untitled](https://s2.loli.net/2022/02/12/kJfRCBnDtoI3uLG.png)

下标17的单独处理 

![Untitled](https://s2.loli.net/2022/02/12/7gYUG4tQV9826L1.png)

而算法这时候对下标17的位置也进行了处理，所以要再处理一次 

![Untitled](https://s2.loli.net/2022/02/12/M4nBzSZ9HK3AID5.png)

修改过后就正确了，不过a4后面又有个位置不一样了 

![Untitled](https://s2.loli.net/2022/02/12/vYGst8KXphQk5DJ.png)

看下a4，其实是跟0x79进行异或

![Untitled](https://s2.loli.net/2022/02/12/vE7rUYsxeRSKFza.png)

0x79在源字符串中的位置就是0x18

![Untitled](https://s2.loli.net/2022/02/12/4Qw9MmT7txlRNZi.png)

其实0x18的位置是-，所以要用-替换

![Untitled](https://s2.loli.net/2022/02/12/gQOmqjY8ti1sKSG.png)

下标35是与0xf进行异或，看下下标34的值是0x62

![Untitled](https://s2.loli.net/2022/02/12/AJcLeZ6BkSofUPm.png)

0x62的值来自于下标为b的位置

![Untitled](https://s2.loli.net/2022/02/12/9WAB3XZmb8wCYus.png)

它存在下面的字符串中

![Untitled](https://s2.loli.net/2022/02/12/UMNYjgZbVOKk7Ph.png)

经过追踪0x22的算法如下

![Untitled](https://s2.loli.net/2022/02/12/cROyFhjtmIUBSWi.png)

![Untitled](https://s2.loli.net/2022/02/12/4NSZYpInKgTl5aq.png)