---
title: ollvm@frida辅助分析ollvm虚假控制流
date: 2022-02-11 16:34:37
tags:
---
# frida辅助分析ollvm虚假控制流

虚假控制流混淆跟指令替换不一样的地方是，指令替换每一个函数块都特别的长，而虚假控制流有很多的x和y，这些都是全局变量

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled.png](https://s2.loli.net/2022/02/11/Zud1ip7enVKAozq.png)

会通过这些全局变量来进行判断,哪些函数会被执行，虚假的不会被执行

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%201.png](https://s2.loli.net/2022/02/11/xJYTN4ASKcvXwCs.png)

这些全局变量可以通过导出函数看到

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%202.png](https://s2.loli.net/2022/02/11/monj4py6wKSBDQk.png)

没有返回值的是虚假控制流，不会被调用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%203.png](https://s2.loli.net/2022/02/11/KmD3XRQTkCxwvLr.png)

str1在GetStringUTFChars处引用，返回值v10传递给了sub_12b44函数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%204.png](https://s2.loli.net/2022/02/11/VcCGQyh8ODtTYaK.png)

hook sub_12b44函数 

参数1是返回值的，在onLeave中打印

![Untitled](https://s2.loli.net/2022/02/11/bAFgDZpXvmQzxqU.png)

sub_12b44函数调用了多次

![Untitled](https://s2.loli.net/2022/02/11/JlUOIkhtNjSFf7Z.png)

修改脚本，调用时打印第2个参数，onLeave时打印第1个参数

![Untitled](https://s2.loli.net/2022/02/11/IkKlG9LtvQbXm13.png)

前面2次分别是传递了str1和str2，第3次是盐字符串

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%208.png](https://s2.loli.net/2022/02/11/YuwE9Ap7gt6zdGo.png)

接着往下看，sub_12b44调用多次

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%209.png](https://s2.loli.net/2022/02/11/q8XYEKVFwb32vn7.png)

然后调用sub_1391c，str1和str2给了v26和v27，传出来的东西给了sub_18ab0

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2010.png](https://s2.loli.net/2022/02/11/xoaXtmbLBc9WUAC.png)

先hook sub_1391c

```jsx
var sub_1391C = base_hello_jni.add(0x1391c);
Interceptor.attach(sub_1391C, {
    onEnter: function(args){
        this.arg0 = args[0];
        console.log("sub_1391C onEnter:",hexdump(args[0]),"\r\n",hexdump(args[1]));
    },onLeave: function (retval) {
        console.log("sub_1391C onLeave:",hexdump(retval),"\r\n",hexdump(this.arg0));
    }
});
```

在函数调用时，参数1是导出的，没什么有用的，参数2是str1，函数执行完之后参数1的值跟str1是一样的了

![Untitled](https://s2.loli.net/2022/02/11/tZP1bQyB9u8CEHM.png)

接着hook sub_18ab0，它有两个参数都是导出的

```jsx
var sub_18AB0 = base_hello_jni.add(0x18AB0);
Interceptor.attach(sub_18AB0, {
    onEnter: function(args){
        console.log("sub_18AB0 onEnter:",hexdump(args[0]),"\r\n",hexdump(args[1]));
    },onLeave: function (retval) {
        console.log("sub_18AB0 onLeave:",hexdump(retval));
    }
});
```

第一个参数是str1，第二个参数是str2,

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2012.png](https://s2.loli.net/2022/02/11/8XY1GeltgywWT6z.png)

返回值跟最终的加密字符串还是不太一样，最终字符串是a2c78374514d...

![Untitled](https://s2.loli.net/2022/02/11/fkdNescCq9XL7S6.png)

返回值看起来像是2个指针，打印出来看一下

```jsx
var sub_18AB0 = base_hello_jni.add(0x18AB0);
Interceptor.attach(sub_18AB0, {
    onEnter: function(args){
        this.arg0 = args[0];
        this.arg1 = args[1];
        console.log("sub_18AB0 onEnter:",hexdump(args[0]),"\r\n",hexdump(args[1]));
    },onLeave: function (retval) {
        console.log("sub_18AB0 onLeave:",hexdump(retval));
        console.log("sub_18AB0 onLeave:",hexdump(retval.readPointer()));
        console.log("sub_18AB0 onLeave:",hexdump(retval.add(Process.pointerSize).readPointer()));
    }
});
```

还是看不出来跟最终的加密字符串有相关

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2014.png](https://s2.loli.net/2022/02/11/xlJWtdRaXh4VZUD.png)

但是sub_18ab0函数传递了两个字符串，它后面调用的函数都只传了一个参数，所以还是继续看这个函数

看下sub_18ab0的反汇编，x1和x0是传递的两个参数，返回值给了x0，但是这里x0是没有用的，这里可以看到x8，x8在arm64里面比较特殊，当返回值的内容大于16个字节时使用x8保存返回内容的地址

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2015.png](https://s2.loli.net/2022/02/11/8bLmpnfEktXisqU.png)

打印一下x8的值  

```jsx
var sub_18AB0 = base_hello_jni.add(0x18AB0);
Interceptor.attach(sub_18AB0, {
    onEnter: function(args){
        this.arg0 = args[0];
        this.arg1 = args[1];
        this.arg8 = this.context.x8;
        console.log("sub_18AB0 onEnter:",hexdump(args[0]),"\r\n",hexdump(args[1]));
    },onLeave: function (retval) {
        console.log("sub_18AB0 onLeave:",hexdump(this.arg8));
    }
});
```

31是buffer，20是长度，0x74ce6853是一个地址

![Untitled](https://s2.loli.net/2022/02/11/1o9XVIRsDNepBMg.png)

打印下0x74ce6853地址看一下

```jsx
var sub_18AB0 = base_hello_jni.add(0x18AB0);
Interceptor.attach(sub_18AB0, {
    onEnter: function(args){
        this.arg0 = args[0];
        this.arg1 = args[1];
        this.arg8 = this.context.x8;
        console.log("sub_18AB0 onEnter:",hexdump(args[0]),"\r\n",hexdump(args[1]));
    },onLeave: function (retval) {
        console.log("sub_18AB0 onLeave:",hexdump(this.arg8));
        console.log("sub_18AB0 onLeave:",hexdump(ptr(this.arg8).add(Process.pointerSize * 2).readPointer()));
    }
});
```

这时可以看到返回值就是最终加密的字符串了，所以sub_18ab0就是核心的算法函数了

![Untitled](https://s2.loli.net/2022/02/11/tdsq3GXe4npY7IA.png)

进入函数看一下参数引用，发现都不存在引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2018.png](https://s2.loli.net/2022/02/11/AjZG9Q5IadDzWN8.png)

看下反汇编，因为x8保存了最终的加密字符串，所以重点看下x8，x8赋值给了x19

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2019.png](https://s2.loli.net/2022/02/11/g3laiAUvbkXSFex.png)

这里x19赋值给了x0,

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2020.png](https://s2.loli.net/2022/02/11/gkfmprRJ2O15tNI.png)

hook sub_19248，打印x0、w2、x1三个参数

```jsx
var sub_19248 = base_hello_jni.add(0x19248);
Interceptor.attach(sub_19248, {
    onEnter: function(args){
        console.log("sub_19248 onEnter:",args[0],args[1],args[2]);
    },onLeave: function (retval) {
        console.log("sub_19248 onLeave:",retval);
    }
});
```

参数输入了两个地址，返回的也是一个地址

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2021.png](https://s2.loli.net/2022/02/11/SXrUOpHJWcsTw2z.png)

dump看一下 ，发现第2个参数就是最终的加密字符串

![Untitled](https://s2.loli.net/2022/02/11/uCfcVF43KRWI597.png)

var_70保存了最终加密字符串

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2023.png](https://s2.loli.net/2022/02/11/lxNnrOG3PaAqVRT.png)

f5一下，看到v11来自v13，v13来自于sub_16900函数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2024.png](https://s2.loli.net/2022/02/11/lKuyDg3XnFp7SJr.png)

hook sub_16900函数，它有3个参数，前面2个是输入，第3个参数是输出，所以也要在函数执行完之后打印一下 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2025.png](https://s2.loli.net/2022/02/11/qlZ8MGictgT1A4I.png)

第3个参数保存的是最终的加密字符串

![Untitled](https://s2.loli.net/2022/02/11/wJ2RyCq9VZgjb4X.png)

看下sub_16900的算法

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2027.png](https://s2.loli.net/2022/02/11/bS7OVlyRK9BoGrP.png)

再进入sub_16530看一下 ，通过之前hook sub_16900可以得知参数的内容，修改一下参数名

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2028.png](https://s2.loli.net/2022/02/11/AeNvC5G1luq9DmR.png)

查看out_buffer引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2029.png](https://s2.loli.net/2022/02/11/P1uUFBXsqil7OfK.png)

看一下sub_16214函数，out_buffer给了v3

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2030.png](https://s2.loli.net/2022/02/11/TAJP5fiWScazDnl.png)

可以看到后面把v4的一部分值给了v3

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2031.png](https://s2.loli.net/2022/02/11/5Bsuw7qVbzyvgQk.png)

sub_15fac引用了v4，所以看一下这个函数 

result的值赋值给了v5

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2032.png](https://s2.loli.net/2022/02/11/jXSncJfdDBezZEN.png)

下面调用了函数sub_1531c并传入了v5

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2033.png](https://s2.loli.net/2022/02/11/sx7UTuNcrLA4af9.png)

进入sub_1531c，可以看到算法，搜一下发现是md5编码

![Untitled](https://s2.loli.net/2022/02/11/rLJOnkbB4vFZuM1.png)

hook sub_1531c的上层函数sub_15fac

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2035.png](https://s2.loli.net/2022/02/11/d6uIEstewmY8K3O.png)

hook 第2个参数

```jsx
var sub_15FAC = base_hello_jni.add(0x15FAC);
Interceptor.attach(sub_15FAC, {
    onEnter: function(args){
        console.log("sub_15FAC onEnter:",ptr(args[1]).readCString());
    },onLeave: function (retval) {
    }
});
```

sub_15fac调用了4次，第2个参数是salt字符串

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2036.png](https://s2.loli.net/2022/02/11/KTWUys7HC8q3mNd.png)

算法总结：

1、多次调用sub_15fac拼接salt字符串

2、md5编码

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%2062dae712e1fb4047898e83766fe1e3dd/Untitled%2037.png](https://s2.loli.net/2022/02/11/cEpv8rQLiTFRusW.png)