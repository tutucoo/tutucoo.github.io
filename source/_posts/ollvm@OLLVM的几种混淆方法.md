---
title: ollvm@OLLVM的几种混淆方法
date: 2022-02-11 15:38:37
tags:
---
# OLLVM的几种混淆方法

## 指令替换

写一个小程序进行测试

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211kq8mx4.png)

导出环境变量，然后使用ollvm进行指令替换

```jsx
export PATH=/home/ollvm/llvm-project-9.0.1/llvm/cmake-build-release/bin:$PATH
clang -mllvm -sub hello_ollvm.c -o hello_ollvm
```

这是原版的 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202117QDGLy.png)

这是经过ollvm指令替换的

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211dDh3qJ.png)

## 虚假控制流

通过添加虚假的控制流程来实现混淆 

```jsx
clang -mllvm -bcf hello_ollvm_bcf.c -o hello_ollvm_bcf
```

其实它只会执行某一个地方，绝大多数的位置都不会执行

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202116jg8yf.png)

## 控制流平坦化

平坦化效果，通常有大量的while循环

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211yOQoyR.png)

执行平坦化指令

```jsx
clang -mllvm -fla hello_ollvm_fla.c -o hello_ollvm_fla
```

平坦化效果

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211LSSV4t.png)

指定函数平坦化混淆

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211ZfpTLA.png)

指定函数不混淆

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211h0q4Nt.png)