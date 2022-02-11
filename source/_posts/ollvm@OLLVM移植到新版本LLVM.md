---
title: ollvm@OLLVM移植到新版本LLVM
date: 2022-02-11 15:25:37
tags:
---
# OLLVM移植到新版本LLVM

由于OLLVM已经好几年没有更新了，最近的更新还是基于LLVM4.0的

进入ollvm github仓库查看文件可以看到修改的位置

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211t33Jkn.png)

下载ollvm

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211nxMT6F.png)

切换到llvm-4.0分支

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Kk1CrI.png)

把Obfuscation目录复制到llvm的Transforms目录下 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211pVXCy6.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Hd5Qb8.png)

修改Transforms/CMakeLists.txt文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211XKN0eh.png)

修改Transform/LLVMBuild.txt文件

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211mhJCkz.png)

进入IPO目录，找到PassManagerBuilder.cpp

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211hZ8ijP.png)

如果路径下的文件不存在，把文件拷贝到路径下就可以了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Nvg1AQ.png)

可以通过查看github上面的文件修改历史知道做了哪些改动

PassManagerBuilder.cpp还添加了下面的代码

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211n3ZKG3.png)

初始化了AesSeed，这些代码都是LLVM没有的，在移植过去的时候需要添加，另外还有一些简单的改动文件，移植的时候注意下就可以了，这里就不一一截图了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211wAGPfj.png)

修改好了以后就可以用ninja LLVMObfuscation编译OLLVM了 

OLLVM有些bug被修复了，根据这些修改对文件进行修改

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211PHJN23.png)

使用ninja clang指令进行编译，如果顺利就移植成功了