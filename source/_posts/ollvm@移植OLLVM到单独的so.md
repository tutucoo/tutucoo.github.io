---
title: ollvm@移植OLLVM到单独的so
date: 2022-02-11 15:30:37
tags:
---
# 移植OLLVM到单独的so

## 概述

移植到单独的so，这样修改OLLVM就不用编译整个LLVM代码

## 整体结构

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211YMpm31.png)

## ollvm目录

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211NsUh5v.png)

include目录下的结构

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211nvHsTY.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/2022021172bFcm.png)

lib目录

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Gzru9L.png)

lib目录下的CMakeLists.txt

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211zMGlhd.png)

lib目录下的LLVMBuild.txt

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211OL3X5P.png)

lib/Transforms目录下

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Ut70Tf.png)

lib/Transforms目录下的CMakeLists.txt

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211gT4jnD.png)

lib/Transforms目录下的LLVMBuild.txt

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211vjmXnQ.png)

lib/Transforms目录下的Obfuscation目录

![image-20220211153136224](https://gitee.com/tutucoo/images/raw/master/uPic/20220211AXPl9G.png)

CMakeLists.txt

![image-20220211153148520](https://gitee.com/tutucoo/images/raw/master/uPic/2022021197F1Ih.png)

## 根目录下CMakeLists.txt

![image-20220211153155749](https://gitee.com/tutucoo/images/raw/master/uPic/20220211tuw8a4.png)

## 用CLion进行编译

只留下build版本，进行编译，生成了libLLVMObfuscation.a文件

![image-20220211153209328](https://gitee.com/tutucoo/images/raw/master/uPic/20220211jjWyAh.png)

给Obfuscation目录下的CMakeLists.txt文件添加一个MODULE标志 

![image-20220211153219001](https://gitee.com/tutucoo/images/raw/master/uPic/20220211kwd393.png)

这时会生成一个LLVMObfuscation.so文件

![image-20220211153239858](https://gitee.com/tutucoo/images/raw/master/uPic/20220211a18OUC.png)

ollvm是通过在PassManagerBuilder.cpp中注册绑定参数

![image-20220211153247761](https://gitee.com/tutucoo/images/raw/master/uPic/20220211oga8TT.png)

在lib/Transforms/Obfuscation目录下创建Entry.cpp

![image-20220211153301860](https://gitee.com/tutucoo/images/raw/master/uPic/20220211dwTVBJ.png)

把Entry.cpp放到Obfuscation目录下

![image-20220211153309077](https://gitee.com/tutucoo/images/raw/master/uPic/2022021136vHl8.png)

添加命名空间

![image-20220211153315511](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Ivsfsp.png)

添加额外的头文件

![image-20220211153322135](https://gitee.com/tutucoo/images/raw/master/uPic/20220211UgxZBs.png)

最后进行注册

![image-20220211153329285](https://gitee.com/tutucoo/images/raw/master/uPic/20220211HJ1dHE.png)

找到之前编译的clang的路径

![image-20220211153336825](https://gitee.com/tutucoo/images/raw/master/uPic/20220211lVE29w.png)

Program arguments填入下面的参数，编译生成ll文件

```jsx
-Xclang -load -Xclang /OLLVM/cmake-build-debug/ollvm/lib/Transforms/Obfuscation/LLVMObfuscation.so -mllvm -fla /home/hello_ollvm_fla.c -emit-llvm -S -o /home/hello_ollvm_fla.ll
```

## 调试ollvm

也可以下断点进行调试

![image-20220211153344796](https://gitee.com/tutucoo/images/raw/master/uPic/20220211vxRnIZ.png)

断点没有断下来，执行一个异常看下代码有没有被执行

![image-20220211153352097](https://gitee.com/tutucoo/images/raw/master/uPic/20220211VT7iPq.png)

程序崩溃，看到个clang-9，这条指令是真正的编译指令 

![image-20220211153400066](https://gitee.com/tutucoo/images/raw/master/uPic/20220211K0Ftkp.png)

看下这个clang工具，发现它有个软链接指向了clang-9

![image-20220211153407036](https://gitee.com/tutucoo/images/raw/master/uPic/20220211amfy2Q.png)

把clang-9的参数拷贝过来，选择clang-9程序

![image-20220211153413647](https://gitee.com/tutucoo/images/raw/master/uPic/20220211sTwiUV.png)

再重新下断点并调试，这时程序正常断下了 

![image-20220211153420188](https://gitee.com/tutucoo/images/raw/master/uPic/20220211bl2joT.png)