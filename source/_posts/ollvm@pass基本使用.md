---
title: ollvm@pass基本使用
date: 2022-02-11 15:19:37
tags: 
---
# pass基本使用

## 概述

pass是LLVM里面重要的框架，不同的pass用于源代码在编译之前进行优化，可以复杂化也可以简单化，ollvm就是pass的一种

下图对bc文件执行了pass，bc文件是机器语言，但是通过—print-bb这个pass，可以打印中间代码

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211u3Xq8m.png)

每个pass都会调用runOnFunction函数，可以对要处理的函数进行修改

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211nsZXyP.png)

## pass使用方法

pass在llvm根目录下lib文件夹的Transforms文件夹中

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211j9yz7h.png)

在进行编译时,目标程序的函数都会传递给runOnFunction函数，这个pass只是打印了函数的名字

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Ksp67O.png)

查看这个pass的CMakeLists.txt文件，添加opt，那么就会生成对应的so文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202116LgN3m.png)

生成的pass名字是LLVMHello.so

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211xho6og.png)

下图是使用自定义pass的命令，-hello是自定义pass定义的参数，可以看到这个pass的功能是打印了一个函数名称main

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211fPzoLR.png)

再给C文件添加个函数test_hello1

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Uupk4q.png)

hello.clang.ll还存在一些外部的函数，比如说printf，它不受pass影响，所以这个函数名没有打印出来，pass只能优化我们自己写的代码 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Wao4vg.png)