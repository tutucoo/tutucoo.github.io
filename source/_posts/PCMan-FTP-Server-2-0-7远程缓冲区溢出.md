---
title: PCMan FTP Server 2.0.7远程缓冲区溢出
date: 2021-01-21 10:54:25
cover: https://gitee.com/tutucoo/images/raw/master/uPic/5SNsfb.jpg
---



# PCMan FTP Server 2.0.7远程缓冲区溢出

## 环境准备

pcman：进入这个地址下载[https://www.exploit-db.com/exploits/26471/](https://www.exploit-db.com/exploits/26471)

python 2.7.0

windows 7 64位 专业版



## 漏洞复现

启动PCMan后，执行poc，PCMan崩溃

```
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.3.163",21))
s.recv(1024)
User = 'anonymous'
Password = 'A'*8000
s.send("USER" + User + "\r\n")
print s.recv(1024)
s.send("PASS" + Password + "\r\n")
print s.recv(1024)
```

![](https://gitee.com/tutucoo/images/raw/master/uPic/NgP7Hx8bhBYK3a6-20210121105455985.png )

## 漏洞分析

用Windbg挂载pcman之后，执行poc，Windbg会断下，可以看到eip的值是41414141，产生了缓冲区溢出

poc中调用了send函数，pcman很可能是通过recv来接收数据的，在IDA中查看recv函数调用的情况，可以看到有两处调用

![](https://gitee.com/tutucoo/images/raw/master/uPic/1730632840271.png)

在Windbg中这两处函数调用下断点，然后执行poc

![](https://gitee.com/tutucoo/images/raw/master/uPic/2828561526913.png)


程序在recv处会断下四次，第四次断下后再执行程序就会产生崩溃，所以需要在第三次断下之后单步进行调试，单步执行完402a26(调用sub_403e60)时程序崩溃，所以需要进入这个函数，接下来通过这种方式一步步靠近最终导致崩溃的函数

第四次在402a26处断下时，dd esp查看堆栈情况
可以看到函数返回地址是402a2b，此时返回地址还是正常的，函数执行完会跳到这个地址继续执行指令
![](https://gitee.com/tutucoo/images/raw/master/uPic/2725184152125.png)

看一下402a2b的地址，正是sub_403e60的下一条指令

![](https://gitee.com/tutucoo/images/raw/master/uPic/47561302131.png)

继续执行到403ee6处，也就是_sprintf函数的位置，在还没有执行_sprintf函数时，dd 18ed68查看堆栈，可以看到函数返回地址仍然是402a2b，执行完_sprintf函数，可以看到函数返回地址已经被冲刷成41414141了，这会造成函数在返回时跳转到41414141接着执行，而41414141是一个非法的地址，程序因此会崩溃，可能是由于_sprintf函数在执行时没有控制源字符串的大小，导致过长的字符串冲刷掉了函数返回地址

![](https://gitee.com/tutucoo/images/raw/master/uPic/431792199120.png )

我们可以通过观察sprintf函数执行前后内存变化来印证这一点
_sprintf函数的第一个参数是目地缓冲区，用于保存拼接的字符串，这个参数保存在寄存器ecx中
![](https://gitee.com/tutucoo/images/raw/master/uPic/5428838106174.png )

查看ecx的地址18E568，这里保存了sprintf函数的10个参数，可以看到第10个参数是超长的字符串
![](https://gitee.com/tutucoo/images/raw/master/uPic/5049600201308.png  )

对应ida里面的a2参数，a2就是传进来的超长字符串
![](https://gitee.com/tutucoo/images/raw/master/uPic/3229776791168.png )

执行完sprintf函数之后，函数返回地址被冲刷了

![](https://gitee.com/tutucoo/images/raw/master/uPic/5632851706756.png )


再接着往下执行，单步到ret 4处，ret 4等同于

```
add esp 4
jmp 返回地址
```

此时返回地址已经被冲刷为41414141了，所以jmp 到返回地址时，程序崩溃

![](https://gitee.com/tutucoo/images/raw/master/uPic/2544833597577.png )


## 总结

通过这个漏洞的复现，我们可以反推漏洞挖掘的过程

通过wireshark抓包（ftp数据包需要选择正确的本地以太网卡，否则ftp数据抓包不全）

可以看到发送一个正常的ftp请求，服务器首先会返回欢迎字样

![](https://gitee.com/tutucoo/images/raw/master/uPic/5917157208251.png )

然后要求输入用户名和密码

![](https://gitee.com/tutucoo/images/raw/master/uPic/2613873151628.png )

![](https://gitee.com/tutucoo/images/raw/master/uPic/4500148821404.png )

对比poc就可以知道为啥poc要这么发了，可以知道在挖掘这个漏洞时，先是通过正常ftp请求，抓包分析，然后通过python脚本模拟请求，成功后再进行fuzz，最后在password后面接上超长字符串，程序崩溃，再通过windbg和ida回溯产生崩溃的函数，最终定位到sprintf函数产生了溢出。

![](https://gitee.com/tutucoo/images/raw/master/uPic/3881747756168.png  )

简单的分析到此结束，谢谢观看～～