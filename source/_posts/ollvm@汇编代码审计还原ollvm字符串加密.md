---
title: ollvm@汇编代码审计还原ollvm字符串加密
date: 2022-02-11 17:16:37
tags:
---
# 汇编代码审计还原ollvm字符串加密

这个例子在搜索.datadiv时没有结果，这是因为修改了默认的ollvm前缀，可以通过查看.init_array段，找到解密字符串函数 

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211u9fTFB.png)

像下图中veorq_s8这种字符串加密，一般只在64位中才有，32位的没有

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%201.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211v2WmvM.png)

这里使用了stru_37010的两个字节

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%202.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211bQgdpk.png)

它在文件中的偏移是0x36010

![image-20220211171710509](https://gitee.com/tutucoo/images/raw/master/uPic/20220211BxEWYX.png)

找到它的位置

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%204.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211NVTQts.png)

选中十六个字节拷贝

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%205.png](%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%205.png)

这十六个字节跟v0进行异或，v0是0xc6

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%206.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211RvfJ75.png)

在010中进行异或操作

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%207.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211wnrH9g.png)

解密后看到这个字符串的值是Hello from JNI

![%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%2014db95f1df144032b242e029ecf69f01/Untitled%208.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211nGGGcw.png)