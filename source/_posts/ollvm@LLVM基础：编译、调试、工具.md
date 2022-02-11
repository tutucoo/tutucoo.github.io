---
title: ollvm@LLVM基础：编译、调试、工具
date: 2022-02-11 15:15:37
tags:
---
# LLVM基础：编译、调试、工具

## LLVM简介

LLVM包含了很多项目 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211AkfVtX.png)

## llvm下载

ndk的llvm/bin目录下，有llvm套件，可以查看这些工具的版本，然后在llvm官网下载相近的版本就可以了

```jsx
./clang --version
```

llvm源码目录

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202117mMtGq.png)

## llvm编译

### 环境准备

进入llvm.org/docs找到llvm文档，按照文档中环境要求进行安装和配置

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211GIDrhI.png)

有个方便的做法是把安卓源码依赖的程序全部安装上，这样基本就满足了llvm编译的环境要求了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211M7Syja.png)

### 编译debug版本

进入llvm根目录创建build_debug目录并进入该目录，执行下面的命令

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211mgCSNc.png)

然后再执行ninja -j8命令

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Q8e6Tn.png)

### 编译release版本

在根目录下创建build_release目录并进入这个目录下，执行下面的命令

-DLLVM_ENABLE_PROJECTS="clang"的意思添加编译clang程序，这样生成的bin目录下就有clang的可执行程序了

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211b015uU.png)

### 使用CLion进行编译

打开项目，路径选择根目录llvm目录下的CMakeLists.txt文件 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211yros28.png)

进入设置，指定编译选项

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211JMuA3C.png)

点击确定之后项目会自动进行build

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211uKw8bl.png)

关闭CLion，进入命令行执行ninja -j8进行编译 

可以给CLion增加内存

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211QlkRNp.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211wC6Esy.png)

也可以进入cmake-build-debug目录单独编译某一个项目，比如说clang

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211yHWUtS.png)

## 测试编译的Clang程序

写一个简单的C文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211sBvRcw.png)

控制台添加刚编译的clang程序的环境变量

```jsx
export PATH=/home/ollvm/llvm-project-9.0.1/llvm/cmake-build-debug/bin:$PATH
```

使用clang对C文件进行编译并运行 

```jsx
clang hello_clang.c -o hello_clang
./hello_clang
```

## 调试编译的Clang程序

找到clang程序  

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211hD618k.png)

设置路径为刚才编辑的c文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211b85gdL.png)

点击确定后，目录下生成了一个可执行文件hello_clang_clion

然后找到clang目录下的tools里面的driver.cpp

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211lWnJdN.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211gEg0ov.png)

在main函数下断点就可以对clang进行调试了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202113lBfwn.png)

## 调试opt程序

找到opt，然后进入配置

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211sf5Tbi.png)

把下图的参数填入Program arguments中

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Xvz3xM.png)

hello_clang.dll文件路径需要修改为绝对路径，如果程序无法运行，可以把so文件路径的单引号去掉试试

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211ze7uCf.png)

opt工具在tools文件夹下

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211sD4R2X.png)

然后在main函数中下断，之后就可以进行调试了

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211T9Af1l.png)

中间文件的源文件也可以下断点进行调试

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211J7A1z7.png)

## llvm工具

### 编译中间语言

编译成中间语言，中间语言可以转为不同平台的汇编语言

```jsx
clang -emit-llvm -S hello_clang.c -o hello_clang.ll
```

生成的ll文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Lri4fu.png)

中间语言文件一样可以被执行

```jsx
lli hello_clang.ll
```

### llvm-as工具

转为汇编之前要先转为bitcode文件，通过llvm-as工具完成

```jsx
llvm-as hello_clang.ll -o hello_clang.bc
```

### llc工具

通过llc工具把bitcode文件转为汇编文件

```jsx
llc hello_clang.bc -o hello_clang.s
```

### llvm-dis工具

通过llvm-dis工具将bitcode文件转为中间文件

```jsx
llvm-dis hello_clang.bc -o hello_clang_re.ll
```

### opt工具

opt工具主要的作用有两个：

- 查看bitcode文件
- 对bc文件或ll文件执行pass