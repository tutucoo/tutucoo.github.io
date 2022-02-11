---
title: 算法@分组密码之AES
date: 2022-02-11 15:10:37
tags:
---
# 分组密码之AES

以AES-128为例，会对明文分组进行10轮迭代运算，加密的第1轮到第9轮的轮函数一样，包括4个操作：字节替换、行位移、列混合和轮密钥加。最后一轮迭代不执行列混合。另外，在第一轮迭代之前，先将明文和原始密钥进行一次异或加密操作

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BAES%208f5210e1a4174f6a8f8795491ae7e026/Untitled.png](https://gitee.com/tutucoo/images/raw/master/uPic/202202111r6VS2.png)

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BAES%208f5210e1a4174f6a8f8795491ae7e026/Untitled%201.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211woepIT.png)

字节代替：字节代替的主要是通过s盒完成一个字节到另一个字节的映射

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Lldjvz.png)

行位移：第一行保持不变，第二行循环左移1个字节，第三行循环左移2个字节，第四行循环左移3个字节 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211b5UQNG.png)

列混淆：主要用于提供AES算法的扩散性，对列混淆矩阵相乘得到结果

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211D1a67h.png)

轮密钥加：每轮的输入与轮密钥异或一次（当前分组和扩展密钥的一部分进行按位异或），因为二进制连续异或一个数结果是不变的，所以在解密时再异或该密钥就可恢复