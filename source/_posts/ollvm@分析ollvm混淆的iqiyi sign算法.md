---
title: ollvm@分析ollvm混淆的iqiyi sign算法
date: 2022-02-11 16:04:37
tags:
---
# 分析ollvm混淆的iqiyi sign算法

sign算法是getQdscJNI函数 

```java
 package com.qiyi;
 
 import java.io.UnsupportedEncodingException;
 
 public class Protect {
 
     public static String getQdsc(Object obj, String str) {
 
         try {
 
             return getQdscJNI(obj, str.getBytes("UTF-8"));
 
         } catch (UnsupportedEncodingException unused) {
 
             return "";
 
         }
 
     }
 
     // getQdsc内部调用的是native函数
 
     private static native String getQdscJNI(Object obj, byte[] bArr);
 
 }
```

通过 frida直接调用getQdsc函数

```java
 function call_getQdsc() {
     Java.perform(function () {
         var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
         var context = currentApplication.getApplicationContext();
         var Protect = Java.use("com.qiyi.Protect");
         var qdsc = Protect.getQdsc(context, "123");
         console.log("qdsc:", qdsc);
     });
 }
```

调用结果

```
 call_getQdsc()
```

主动调用qdsc函数生成的字符串 

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled.png](https://gitee.com/tutucoo/images/raw/master/uPic/202202118DpuVS.png)

IDA加载`libprotect.so`，找到函数getQdscJNI

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%201.png](https://gitee.com/tutucoo/images/raw/master/uPic/202202111D0cUX.png)

先来看一个简单的ollvm 流程平坦化之后的流程图是怎么样的

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%202.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211bSPORR.png)

进入sub_126B4，看下整体流程图

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%203.png](https://s2.loli.net/2022/02/11/GK3nHveY9Lq4w1o.png)

先来看看函数定义，由前面的调用参数可以知道a1是JNIEnv的指针，a3是一个char*的buffer，a4是buffer的长度

```
 unsigned int *__fastcall sub_126B4(JNIEnv *a1, int a2, char *buffer, int buffer_len)
```

第一步找到buffer的交叉引用

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%204.png](https://s2.loli.net/2022/02/11/wlBxLcZeGIdbkYf.png)

对v7 重命名为 buffer_1， 查看buffer_1的交叉引用，看到原始数据传入了sub_1189C

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%205.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211cFW3Po.png)

查看sub_1189C

```
 v70 = (unsigned int *)sub_1189C(&v103, buffer_1, v8);
```

查看v8的引用,看到v8被参数buffLen赋值

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%206.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211cnHYyc.png)

v8就是bufferLen， 对v8重命名为bufferLen_

sub_1189C 的后两个参数一个buffer, 一个是len, 因此先写一段frida脚本来hook sub_1189C函数

```jsx
 function hook_native() {
     var base_protect = Module.findBaseAddress("libprotect.so");
     if (base_protect) {
         console.log("base_protect:", base_protect);
         var sub_1189C = base_protect.add(0x1189C);
         console.log("sub_1189C:", sub_1189C);
         Interceptor.attach(sub_1189C, {
             onEnter: function (args) {
                 this.buffer = args[1];
                 this.len = parseInt(args[2]);
                 console.log("sub_1189C onEnter:\n", hexdump(this.buffer, { length: this.len }));
             },
             onLeave: function (retval) {
             }
         });
     }
 }
```

调用call_getQdsc, 可以看到sub_1189C的参数buffer已经打印出来了，可以看到这时的buffer不再是123,而是aXFpqiyi.Protec=

```
 
 
 [Google Pixel XL::com.qiyi.video]-> hook_native()
 
 base_protect: 0xd0a64000
 
 sub_1189C: 0xd0a7589c
 
 undefined
 
 [Google Pixel XL::com.qiyi.video]-> call_getQdsc()
 
 sub_1189C onEnter:
 
 0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF
 
 c1505aa0 61 58 46 70 71 69 79 69 2e 50 72 6f 74 65 63 3d aXFpqiyi.Protec=
 
 sub_1189C onEnter:
 
 0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF
 
 d160bd00 33 30 38 32 30 32 34 37 33 30 38 32 30 31 62 30 30820247308201b0
 
 d160bd10 61 30 30 33 30 32 30 31 30 32 30 32 30 34 34 63 a00302010202044c
 
 d160bd20 39 64 61 36 61 30 33 30 30 64 30 36 30 39 32 61 9da6a0300d06092a
 
 d160bd30 38 36 34 38 38 36 66 37 30 64 30 31 30 31 30 35 864886f70d010105
 
 d160bd40 30 35 30 30 33 30 36 37 33 31 30 62 33 30 30 39 05003067310b3009
 
 d160bd50 30 36 30 33 35 35 30 34 30 36 31 33 30 32 34 33 37af87
 
 qdsc: 616904a6660256fafef2db967eb82880
```

补充一下，对sub_126b4进行hook，传进来123，123是怎么变成aXFpqiyi.Protec=的？

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%207.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211yrQb2F.png)

暂时先不分析sub_1189C函数，先把sub_126B4的buffer参数引用分析完。

选择第三个引用

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%208.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211pEvvbe.png)

再次查看ptr的交叉引用

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%209.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211bN2xsK.png)

查看sub_11A68，两个参数一个buffer, 一个是len, 写一段frida脚本hook sub_11A68

```jsx
 
 
 var sub_11A68 = base_protect.add(0x11A68);
 Interceptor.attach(sub_11A68, {
	 onEnter: function(args) {
		 this.buffer = args[2];
		 this.len = parseInt(args[3]);
		 console.log("sub_11A68 onEnter:\n", ptr(this.buffer).readCString(this.len)); 
 },
	 onLeave: function(retval) {}
 });
```

现在把引用buffer的几个函数都已经找到了，下面深入分析sub_1189C、sub_11A68这两个函数的功能

```
 
 
 _BYTE *__fastcall sub_1189C(int a1, char *a2, int a3)
 
 {
 
 //省略部分变量定义代码
 
 v3 = a1;
 
 v4 = a3;
 
 v5 = a2;
 
 v6 = malloc(0x21u);
 
 _aeabi_memclr(v6, 33);
 
 v21 = 1732584193;
 
 v22 = 4023233417;
 
 v23 = 2562383102;
 
 v24 = 271733878;
 
 v25 = 0;
 
 v26 = 0;
 
 sub_11A68(v3, (int)&v21, (int)v5, v4);
 
 v7 = v25;
 
 v8 = (v25 >> 3) & 0x3F;
 
 *((_BYTE *)&v27 + v8) = 0x80u;
 
 v9 = v8 ^ 0x3F;
 
 v10 = (char *)&v27 + v8 + 1;
 
 if ( v9 > 7 )
 
 {
 
 _aeabi_memclr(v10, v9 - 8);
 
 }
 
 else
 
 {
 
 v11 = _aeabi_memclr(v10, v9);
 
 sub_11CAC(v11, &v21, &v27);
 
 _aeabi_memclr4((int)&v27, 56);
 
 v7 = v25;
 
 }
 
 v28 = v7;
 
 v29 = v26;
 
 sub_11CAC(v26, &v21, &v27);
 
 v17 = v21;
 
 v18 = v22;
 
 v19 = v23;
 
 v20 = v24;
 
 _aeabi_memclr4((int)&v21, 88);
 
 v12 = 0;
 
 do
 
 {
 
 v13 = *((unsigned __int8 *)&v17 + v12);
 
 v14 = a0123456789abcd[v13 >> 4];
 
 LOBYTE(v13) = a0123456789abcd[v13 & 0xF];
 
 v6[2 * v12] = v14;
 
 v15 = (int)&v6[2 * v12++];
 
 *(_BYTE *)(v15 + 1) = v13;
 
 }
 
 while ( v12 != 16 );
 
 result = (_BYTE *)(_stack_chk_guard - v30);
 
 if ( _stack_chk_guard == v30 )
 
 result = v6;
 
 return result;
 
 }
```

在算法分析过程中，首先看到类似第`37，38， 39，40行`这种的常量

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2010.png](https://gitee.com/tutucoo/images/raw/master/uPic/202202112HYrSC.png)

如果你已经熟悉这个常量，就能猜到这个是什么算法，通过在算法文件中搜索67452301之后，发现md5, sha1都有`67452301, EFCDAB89, 98BADCFE, 10325476`这四个常量，而sha1多一个`C3D2E1F0`

因此可以判断出`sub_1189C是一个与md5相关的函数`

先对一些变量进行重命名

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2011.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211hdK4LI.png)

进去 `sub_11A68` 看看

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2012.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211mImKjZ.png)

sub_11A68函数里面对参数进行了一些计算，然后调用了sub_11CAC函数，在算法逆向过程中，遇到一些移位异或运算，先粗略看看，然后跳过，只有在算法还原的过程中才仔细分析这些运算

进去`sub_11CAC`看看

![image-20220211160742433](https://s2.loli.net/2022/02/11/vakwigL45Z3uGzT.png)

把sub_11CAC函数中的用到常量与md5_process函数的常量比较发现，都是一样的。现在可以把sub_11CAC命名为md5_process了

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2014.png](https://s2.loli.net/2022/02/11/dZwMuvA4RJHYtV8.png)

确定了md5_process函数之后，就可以确定它的上层函数sub_11A68为md5_update函数，

sub_11A68的上层函数sub_1189C这时候看起来就是一个md5函数的结构了

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2015.png](https://s2.loli.net/2022/02/11/fW3Njx1TgYKbr6S.png)

frida打印出sub_1189C的返回值

```jsx
 function hook_native() {
	 var base_protect = Module.findBaseAddress("libprotect.so");
	 var sub_1189c = base_protect.add(0x1189c); 
	 Interceptor.attach(sub_1189c, {
		 onEnter: function(args) {
			 this.buffer = args[1];
			 this.len = parseInt(args[2]);
			 console.log("sub_1189c onEnter:\n", ptr(this.buffer).readCString(this.len));
 },
		 onLeave: function(retval) {
			 console.log("sub_1189C onLeave retval:\n", hexdump(retval, { length: 0x20 }));
 }
 });
```

aXFpqiyi.Protec=经过md5变成25c11bfb902a449f8d636900a3405419

```
 sub_1189C onEnter:
 
 0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF
 
 c9463e48 61 58 46 70 71 69 79 69 2e 50 72 6f 74 65 63 3d aXFpqiyi.Protec=
 
 sub_1189C onLeave retval:
 
 0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF
 
 c62dd318 32 35 63 31 31 62 66 62 39 30 32 61 34 34 39 66 25c11bfb902a449f
 
 c62dd328 38 64 36 33 36 39 30 30 61 33 34 30 35 34 31 39 8d636900a3405419
```

看完1189c再通过buffer交叉引用找到ptr，再通过ptr的交叉引用找到md5_update，之前已经找到所有buffer引用的地方，一个是1189c，还有一个11a68，1189c已经分析完了，然后要接着向下分析

再来看看ptr的交叉引用，没看到引用的函数

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2016.png](https://s2.loli.net/2022/02/11/KwsEAxG9bBJmOhX.png)

再看看v103和v104的交叉引用

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2017.png](https://s2.loli.net/2022/02/11/XBmMD2HeIJ5LjTc.png)

进入sub_11B64函数看下

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2018.png](https://s2.loli.net/2022/02/11/WM4IFirtgUTvR1w.png)

继续写frida脚本，hook sub_11B64

```jsx
 
 var sub_11B64 = base_protect.add(0x11B64);
 Interceptor.attach(sub_11B64, {
	 onEnter: function(args) {
		 this.arg0 = args[0];
		 this.arg1 = args[1];
		 this.arg2 = args[2];
	 },
	 onLeave: function(retval) {
		 console.log("sub_11B64 onLeave:\n", hexdump(this.arg1, { length: 0x10 }));
	 }
 });
```

可以看到sub_11B64的a2输出结果和Java层调用qdsc函数的结果一样

```
 [Google Pixel XL::com.qiyi.video]-> call_getQdsc()
 
 
 sub_11B64 onLeave:
 
 0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF
 
 de51cab0 61 69 04 a6 66 02 56 fa fe f2 db 96 7e b8 28 80 ai..f.V.....~.(.
 
 
 qdsc: 616904a6660256fafef2db967eb82880
```

此时已经分析出结果

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2019.png](https://s2.loli.net/2022/02/11/ECkLdVFwunUj4yZ.png)

通过hook sub_11a68 也就是md5_update，可以看到多次调用sub_11a68

![%E5%88%86%E6%9E%90ollvm%E6%B7%B7%E6%B7%86%E7%9A%84iqiyi%20sign%E7%AE%97%E6%B3%95%20a3693b6ff722425198cd087b55e69b00/Untitled%2020.png](https://s2.loli.net/2022/02/11/TDrpVws3auCkge9.png)

qdsc = md5(str + "0n9wdzm8pcyl1obxe0n9qdzm2pcyf1ob");

分析被Olllvm混淆的算法的时候，如果IDA能够F5反编译出来，那么按下面的思路来分析算法:

从参数找交叉引用，通过frida hook使用参数的地方(函数)。

如果还没分析出算法，再从结果找交叉引用，用frida hook返回结果的地方(函数)进行分析