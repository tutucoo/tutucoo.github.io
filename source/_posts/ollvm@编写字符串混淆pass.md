---
title: ollvm@编写字符串混淆pass
date: 2022-02-11 15:41:37
tags:
---
# 编写字符串混淆pass

字符串混淆pass原理是编译时先混淆，使用前再解混淆

创建一个StringObf.h

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211u9bAJE.png)

创建一个StringObf.cpp

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211cddyjm.png)

加入到CMakeLists.txt中 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211h9JTf7.png)

在Entry.cpp中添加StringObf.h并绑定变量

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202110OUQqd.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211VQnUNH.png)

此时编译一下项目应该是编译成功的

创建一个测试文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211z5pPsa.png)

声明环境变量，使用clang编译测试文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211fNsMj4.png)

测试文件的IR代码

@是全局变量的标识

%是局部的寄存器

可以看到这个例子里有全局的str，也有%str

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211IRyjxY.png)

我们可以先对字符串进行逐位异或加密，再在使用前进行解密

要先进行加密就要对字符串进行赋值

写一个测试语句

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211jkm2k7.png)

看下IR中间代码，这里将y保存到下标arrayidx处，arrayidx来自%0的第0个，%0来自字符串数组

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211HbZSon.png)

在StringObf.cpp中添加如下代码，先枚举所有基本块，再枚举所有指令

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/2022021130sfGH.png)

执行编译，生成pass，使用这个pass编译测试文件

```jsx
clang -Xclang -load -Xclang /home/OLLVM/cmake-build-debug/ollvm/lib/Transforms/Obfuscation/LLVMObfuscation.so -mllvm -strobf hello_ollvm_7.c -emit-llvm -S -o hello_ollvm_strobf.ll
```

指令会在控制台中被打印出来 

![image-20220211154531275](https://gitee.com/tutucoo/images/raw/master/uPic/202202114XfcV3.png)

打印出每条指令之后再拿到指令里面的值，我们的目标是找@str，所以过滤str字符串

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211XHsbo5.png)

判断是否是一个全局变量，如果是全局变量可以通过判断进入内部逻辑，要找的就是全局变量@str

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211t28aSm.png)

找到全局str，获取到它的值 

![image-20220211154546242](https://gitee.com/tutucoo/images/raw/master/uPic/20220211K6WNns.png)

接着获取到一个异或的值对字符串进行加密 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211GDpjny.png)

接着要进行解密，进行解密需要先申请一个数组，为了了解llvm里面的数组是如何申请的，先在源码里申请一个数组然后进行编译看IR指令 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202112Uyic0.png)

alloca[7*i8]里面的7是数组的长度，先创建了一个数组，再通过bitcast拿到它的指针，这个bitcast就是llvm的函数 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211JBz0dT.png)

用AllocaInst创建一个数组，数组的名字随便拼接一个就可以，最后一个参数是插入当前指令之前

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211urQ7Xg.png)

如果不会使用可以在llvm项目里面搜关键字看下用法

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202113grx7e.png)

创建数组在IR指令中是这样的，发现指令不对，还差一个对齐，这样就需要找到AllocaInst另一个API

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211zK5VKv.png)

至于有哪些API，在编写代码中点击进行看头文件直接可以看到

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202119OALXV.png)

添加对齐参数，并且使用数组ArrayType

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211o9xO1l.png)

这个时候创建成功

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211b34qvc.png)

api使用方法有以下几种：

1、可以通过在编写源码的时候跳转看api

2、可以在llvm项目里面搜索关键字看看llvm是怎么用的

3、llvm官方文档查看使用方法

接着拿到数组的指针

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202115IR8Vo.png)

再获得字符串数组里面每个下标指针

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202117kyNFj.png)

需要注意要调用insertBefore，否则这里就是反顺序的 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202111h8Lmn.png)

再添加一个逻辑，把字符串数据保存到刚才创建的数组里 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Zl0wdf.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211nVZ5sz.png)

可以把指令的值全部用%.str.bitcast代替

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211wBgiXS.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Bneknu.png)

代替前

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211A2dTVe.png)

代替后

![Untitled](%E7%BC%96%E5%86%99%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86pass%20752ec966d0804194a1abb60834323e3c/Untitled%2032.png)

再加入异或的代码

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Ze6xXj.png)

异或后进行保存

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211fXDxHq.png)

要提前保存下异或的key 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211slc3Ze.png)

在使用前进行解密

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202116m1D3F.png)

可以看到每次在存储前进行异或解密，这里的xor的key是10进制的98

![image-20220211154614212](https://gitee.com/tutucoo/images/raw/master/uPic/20220211cAHdiK.png)

这样就无法看到原字符串的值，但是这样的静态反混淆是不够的，完全可以对这些字符进行解密，直接进行异或计算就可以得到原来的字符串

在优化编译的时候也是可以直接还原的

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211IuUmt4.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211nidHiZ.png)

为了防止被优化，需要创建个变量进行中转

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211e1DN1D.png)

将把key保存到另一个变量，再取出，最后使用这个变量进行操作

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202110YHyqK.png)

可以看到这里是对%0进行操作

![image-20220211154627516](https://gitee.com/tutucoo/images/raw/master/uPic/202202111AppLq.png)

平坦化结合字符串加密

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211tGnOXT.png)

虽然这样增加了逆向的难度，这里的字符串也被分割成了字符，而且无法一眼看出原字符串

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211gL1tIF.png)

在汇编视图可以看到xor指令，这样还是可以还原字符串，只是过程比较繁琐 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211rq4cms.png)