---
title: 算法@base64算法
date: 2022-02-11 14:36:41
tags:
---


# base64算法

下图是对字符串“BC”进行base64编码 

首先对BC进行二进制分解，分解为8位2进制，接着每6位2进制分为一组，末尾不足的位补0，最后按照ASCII编码表还原字符串，按照6位一组，最少4组，不足的用=号补位

![Untitled](https://s2.loli.net/2022/02/11/LdSv3RN6GJ7scEZ.png)

