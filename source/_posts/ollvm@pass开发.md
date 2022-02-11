---
title: ollvm@pass开发
date: 2022-02-11 14:51:37
tags:
---

# pass开发

## 在llvm源码之外开发pass

创建下面几个文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211H0mGNS.png)

根目录下的CMakeLists.txt文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211PG01ap.png)

EncodeFuctionName目录下的CMakeLists.txt

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211G6hiA7.png)

然后用CLion打开，有报错，解决报错

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211qwg45s.png)

修改根目录下CMakeLists.txt

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211QnFj3W.png)

还缺这两个文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211J2nV4H.png)

llvm编译目录下进行搜索找到这两个文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211MDJwlf.png)

获取完整路径 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211wGNg87.png)

设置llvm目录

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211dYhlQ3.png)

再声明子目录

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202113viXB2.png)

如果重名报错修改一下文件夹名称就可以了

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211lbu08J.png)

这里添加一个release就可以编译一个release版本

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202111jM2ro.png)

编译好之后会生成LLVMEncodeFunctionname2.so文件，使用opt指令就可以进行使用了

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Jfrl48.png)

## 注册pass

每次加载pass要指定绝对路径还是比较麻烦的，可以通过注册pass，这样就不用每次指定绝对路径

找到include目录 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211ez0Cpo.png)

创建一个头文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211x8zKnx.png)

创建一个接口

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/2022021105Iwac.png)

包含头文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211jLkgG5.png)

在EncodeFunctionName.cpp实现该接口

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211hA9Jui.png)

添加一些实现

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211gmCZYu.png)

判断一下

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211WPKlkD.png)

找到PassManagerBuilder.cpp添加EncodeFunctionName.h头文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211R9qBBu.png)

在CMakeLists.txt中添加下面的指令以便编译成静态库

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211e8qksM.png)

在llvm/lib/Transforms/LLVMBuild.txt中添加到common中

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211QEuDMr.png)

llvm/lib/Transforms/IPO/LLVMBuild.txt中也要添加

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202118icuDM.png)

创建LLVMBuild.txt文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211gzStrj.png)

添加参数

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211hiIyhW.png)

还是在PassManagerBuilder.cpp中添加代码，如果命令执行时有参数就会执行这个pass

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Uodg9n.png)

然后执行ninja LLVMEncodeFunctionName单独进行编译

再编译clang工具 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211rlLL9G.png)

这时就可以把自定义的pass当作参数传递进去进行编译了

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211VXO978.png)

## 在llvm源码外调试pass

这里路径修改为debug版本的 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211DAhDrB.png)

找到opt的绝对路径

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211FXKcLY.png)

在CLion中进行指定

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202115TFRTJ.png)

参数填写

```php
-load pass绝对路径 -pass参数 ll文件绝对路径
```

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202111XiJbD.png)

然后在runOnFunction里面下断点进行调试

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211LqqAv5.png)

## clang加载pass

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211tD3Wva.png)