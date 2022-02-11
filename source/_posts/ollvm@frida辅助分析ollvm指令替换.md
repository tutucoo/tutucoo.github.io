---
title: ollvm@frida辅助分析ollvm指令替换
date: 2022-02-11 16:20:37
tags:
---
# frida辅助分析ollvm指令替换

jni加密算法sign2

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled.png](https://s2.loli.net/2022/02/11/K5Cus7PxkzdYXpm.png)

主动调用sign2

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%201.png](https://s2.loli.net/2022/02/11/UKeEO2unG1h9APZ.png)

返回的sign有时候一样，有时候不一样

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%202.png](https://s2.loli.net/2022/02/11/WI1N3XhPkjO9iTx.png)

f5失败，可以加载jni.h文件，也可以通过加载类型库的方式解决

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%203.png](https://s2.loli.net/2022/02/11/E9GrhMvBXCJ8iyj.png)

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%204.png](https://s2.loli.net/2022/02/11/cUnvW25NksMAF8y.png)

看下JNIEnv的引用，v7改为_env

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%205.png](https://s2.loli.net/2022/02/11/21drWsl6wkOSeZV.png)

接着hook sign2函数，看下GetStringUTFChars的参数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%206.png](https://s2.loli.net/2022/02/11/1UWB5nFedkamghG.png)

sign2的参数和返回值都打印出来了

```jsx
sign2 str1: 0123456789
sign2 str2: abcdefg
sign2 retval: a2c78374514d7934432cbc66cd1e293b
```

先从sign2函数的str1参数开始分析，看到两处引用，赋值给了v5

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%207.png](https://s2.loli.net/2022/02/11/5QNzY7VDJ6naWrl.png)

将v5改名为str_1_1，v8改为c_str_1

看下str_1_1的引用，发现是调用ReleaseStringUTFChars函数时使用的，所以接着看c_str_1

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%208.png](https://s2.loli.net/2022/02/11/RicCdrHoIyUeJ2z.png)

看下c_str_1的引用，调用了memcpy，赋值给了v11

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%209.png](https://s2.loli.net/2022/02/11/U2vNXmsfREOeYMu.png)

v11引用，v11在c_str_1赋值后，后续没有再使用，所以分析v161

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2010.png](https://s2.loli.net/2022/02/11/4sFvcwoZ7CK6L8H.png)

v161引用，203行之前的先不看，先看后面的

v161赋值给了v75，v75不是一个指针，先不看

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2011.png](https://s2.loli.net/2022/02/11/H6iNjOlMJgcsYq3.png)

v161赋值给了v154，然后跳转到了LABEL_22

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2012.png](https://s2.loli.net/2022/02/11/uA5etRabYTDKj6w.png)

看下v154的引用，看下sub_1DFB4

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2013.png](https://s2.loli.net/2022/02/11/d8WbRYEt2qDulBN.png)

hook sub_1dfb4

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2014.png](https://s2.loli.net/2022/02/11/AwVSPUHagrMEvq5.png)

sub_1dfb4的两个参数和返回值，看起来像个地址

```jsx
sub_1DFB4 0x74b5cc6b90 0x74b5cc6b70
sub_1DFB4 onLeave 0x74ce6855d0
```

dump这些地址的内容

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2015.png](https://s2.loli.net/2022/02/11/ZqhBs7lVrexo25P.png)

sub_1dfb4的第一个参数和第二个参数是传递进来的str1和str2

![Untitled](https://s2.loli.net/2022/02/11/Tq23yM1pLzJsjow.png)

函数执行完返回的值是加密的，可以看出虽然sub_1dfb4反编译的时候没有返回值，但其实有返回值的 

可以看到sub_1dfb4的返回值跟最终的加密sign是一致的

![Untitled](https://s2.loli.net/2022/02/11/MtIaiN7rZXojcBG.png)

在sub_1dfb4上面，有一堆指令替换的操作，本来很简单的指令弄的很复杂，这些都不用看，直接进入sub_1dfb4进行分析

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2018.png](https://s2.loli.net/2022/02/11/H9ljMOKx5Z7BN1E.png)

进入sub_1dfb4，参数又变成了一个，这个不用管，在函数名上面右键Set item type，可以设置多个参数以及类型

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2019.png](https://s2.loli.net/2022/02/11/ZutN4j13favW27H.png)

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2020.png](https://s2.loli.net/2022/02/11/NwCoJOb6ETB9vFI.png)

查找两个参数的引用，发现是空

查看反汇编，并没有看到参数的痕迹，x0和x1并没有被使用，但是调用了sub_1e298，可能参数会直接传递到这个函数里面

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2021.png](https://s2.loli.net/2022/02/11/DfRI9dp6mx5ao4b.png)

hook sub_1e298

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2022.png](https://s2.loli.net/2022/02/11/Tbz8lNcCBxRSq4D.png)

可以看到sub_1e298的参数就是str1和str2

![Untitled](https://s2.loli.net/2022/02/11/AhwTKHvEGo4JlaO.png)

先看一下参数1的引用，它的值给了v16和v17

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2024.png](https://s2.loli.net/2022/02/11/ZD5tJH4u7oeEbSn.png)

然后看v16的引用，没有什么有价值的

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2025.png](https://s2.loli.net/2022/02/11/dTwn7iFW8DY6lUG.png)

再看v17的引用，可以看到v17赋值给了v20

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2026.png](https://s2.loli.net/2022/02/11/mjJLDFRNlokCzBi.png)

v20之前的值来自v28或者v5

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2027.png](https://s2.loli.net/2022/02/11/dUghjsp4R8xGq6z.png)

先看下v28，v28没有什么东西，看下v5

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2028.png](https://s2.loli.net/2022/02/11/ApdaMqnIEsbiwHt.png)

v5里面调用了一个函数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2029.png](https://s2.loli.net/2022/02/11/hmFaoCpOwlMnJVU.png)

可以看到v5两处调用，因为sub_1ab0上面调用了memcpy，把str1赋值给了v20，所以下面要对sub_1ab50进行hook，看str1有没有传递

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2030.png](https://s2.loli.net/2022/02/11/JW63tRalKoe8bQx.png)

hook它的三个参数和返回值 

```jsx
var sub_1AB50 = base_hello_jni.add(0x1AB50);
Interceptor.attach(sub_1AB50, {
    onEnter: function(args){
        console.log("sub_1AB50 onEnter:",(args[0]),"\r\n",(args[1]),args[2]);
    },onLeave: function (retval) {
        console.log("sub_1AB50 onLeave:",retval);
    }
});
```

两次调用了sub_1ab50，参数1和参数2是个指针，参数3应该是个长度

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2031.png](https://s2.loli.net/2022/02/11/j6OxaL3FZy8cGYW.png)

dump一下地址，可以看到这个函数返回的内容

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2032.png](https://s2.loli.net/2022/02/11/qoyu3d6Qc7pfHSA.png)

第二次调用是传递的加盐字符串

![Untitled](https://s2.loli.net/2022/02/11/vwJWDUOt6rAeFZq.png)

返回了加盐拼接字符串

```jsx
++++++++++++salt2+
```

所以可以认为sub_1ab50函数的功能主要是拼接字符串的

hook sub_1e298的返回值

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2034.png](https://s2.loli.net/2022/02/11/k3SlnwJTLrOjhER.png)

可以看到也是返回的拼接字符串

![Untitled](https://s2.loli.net/2022/02/11/rUYjIcRuTX5hHEl.png)

看下sub_1e298的返回值X0，虽然X0没有被使用，不过下面调用了sub_1ab4c

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2036.png](https://s2.loli.net/2022/02/11/oynZ59Ae8vO6rUx.png)

反编译没有返回值，但是有三个参数，其中最后一个是返回值 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2037.png](https://s2.loli.net/2022/02/11/OURi9KlhWwLkIYm.png)

hook sub_1ab4c

```jsx
var sub_1AB4C = base_hello_jni.add(0x1AB4C);
Interceptor.attach(sub_1AB4C, {
    onEnter: function(args){
        console.log("sub_1AB4C onEnter:",args[0],(args[1]),args[2]);
    },onLeave: function (retval) {
        console.log("sub_1AB4C onLeave:",retval);
    }
});
```

hook结果 

```jsx
sub_1AB4C onEnter: 0x74b5cc6ad9 0x11 0x74b5cc6b18
sub_1AB4C onLeave: 0x74b5cc6960
sign2 retval: 43898b3fe54cfe43af19206c592acaf3
```

sub_1ab4c第一个参数传递了拼接字符串，其他的参数和返回值没有有用的 

![Untitled](https://s2.loli.net/2022/02/11/DRAHzrTn8ONkKLV.png)

因为它第三个参数可能是个返回值，所以函数执行完以后打印这个参数看下，可以看到这第三个参数确实就是最终的加密字符串

![Untitled](https://s2.loli.net/2022/02/11/yckudsmgqaCvhVG.png)

sub_1ab4c就是核心的加密函数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2040.png](https://s2.loli.net/2022/02/11/SgvmrezJUEP2slM.png)

它的内部调用了sub_195bc

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2041.png](https://s2.loli.net/2022/02/11/PzlEVZkbWm2d9OK.png)

首先看下input_buffer的引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2042.png](https://s2.loli.net/2022/02/11/7YbPZSlhHdVxgnW.png)

v3改名为_input_buffer，再看_input_buffer的引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2043.png](https://s2.loli.net/2022/02/11/Rmy56Pp7YbtIUcf.png)

hook sub_171c4函数，第1个参数有返回值 ，第2个参数是拼接字符串

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2044.png](https://s2.loli.net/2022/02/11/EnyiVkFALfU7wI2.png)

看到都是指针，第一个参数跟返回值都是一样的地址

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2045.png](https://s2.loli.net/2022/02/11/DGhjzkbv9l84Fm5.png)

hexdump看一下，第1个参数和返回值都是40，第2个参数是拼接字符串

![Untitled](https://s2.loli.net/2022/02/11/RlkuUFdwyNVYc4p.png)

![Untitled](https://s2.loli.net/2022/02/11/wV6XWzrs3bnDB9M.png)

因为没有看到最终的加密字符串，进入sub_171c4函数看下

看到常量，搜索一下，搜索出来的结果显示这里是md5编码

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2048.png](https://s2.loli.net/2022/02/11/VeCiOxslh6U5QPD.png)

可以看到对sub_171c4的第二个参数进行md5编码可以得到最终的加密字符串

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2049.png](https://s2.loli.net/2022/02/11/tl1NOwTGkKH8BpL.png)

算法总结：

1、0123456789拼接abcdefg

2、添加盐

3、md5运算

之所以每次加密的字符串都不一样是因为盐字符串每次都会变化

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2050.png](https://s2.loli.net/2022/02/11/fxDFlqsYpOjegrZ.png)

查找盐字符串生成的函数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2051.png](https://s2.loli.net/2022/02/11/aKIEpNzMdRvkbnV.png)

可以看到是sub_1ab50返回的 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2052.png](https://s2.loli.net/2022/02/11/s4FYB5foe1UOQJn.png)

sub_1ab50内部调用了随机函数生成随机的盐字符串

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8C%87%E4%BB%A4%E6%9B%BF%E6%8D%A2%2013ef8eb0f39a411fb74779cf8a6a7e2d/Untitled%2053.png](https://s2.loli.net/2022/02/11/Z1DTWUwVEG6KceX.png)