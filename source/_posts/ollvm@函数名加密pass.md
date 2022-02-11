---
title: ollvm@函数名加密pass
date: 2022-02-11 15:40:37
tags:
---
# 函数名加密pass

在Transforms下面创建EncodeFunctionName目录，创建一个EncodeFunctionName.cpp文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211iOaOmt.png)

创建下图中的几个文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211IaLr3T.png)

Reload CMake Project

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211OiWHES.png)

把新创建的pass添加到Transforms目录下的CMakeLists.txt文件中

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211gBp2w7.png)

然后在EncodeFunctionName.cpp中添加头文件和命名空间

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211uj4gYq.png)

然后根据官方文档把下面的代码添加上去

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211pyGKbU.png)

进入cmake-build-release目录，直接单独编译这个模块 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211O3gnpL.png)

找到pass路径

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211HdJCgu.png)

通过pass对ll文件进行处理生成bc文件，可以看到pass文件里面的语句被执行了，pass成功编译进去了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211zyNbSL.png)

对函数名进行修改

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211i027Xc.png)

运行时函数名还是原来的，只是静态反编译的情况下会被混淆

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211fHCuPS.png)