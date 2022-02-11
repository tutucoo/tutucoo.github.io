---
title: ollvm@移植OLLVM到NDK中
date: 2022-02-11 15:35:37
tags:
---
# 移植OLLVM到NDK中

进入llvm目录，复制三个目录到NDK的linux-x86_64目录下，覆盖掉

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211YC0w65.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211y1T1CQ.png)

创建个c++项目

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202116vpkRu.png)

默认的项目会使用自带的NDK，有的时候我们需要使用下载下来的ndk，只要在配置文件里指定一下ndk的目录就可以了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211591BRt.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211NoXyzy.png)

添加ollvm声明，编译时会加入这些参数 

![image-20220211153712292](https://gitee.com/tutucoo/images/raw/master/uPic/20220211GQ2qqr.png)

写一个示例代码，太简单的话基本块很少，没法分割进行混淆

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211xG3YYm.png)

编译成汇编

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211SJgBpL.png)

生成的在下面的目录里面

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202116eynt1.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211bKmHt0.png)

可以看到代码已经被混淆了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202118EP3h1.png)