---
title: ollvm@通过frida hook还原ollvm字符串加密
date: 2022-02-11 16:49:37
tags:
---
# 通过frida hook还原ollvm字符串加密

app在OnCreate中调用了jni函数sign1

![%E9%80%9A%E8%BF%87frida%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%205c6fc7e7648b438bb22a21f93c41641f/Untitled.png](https://s2.loli.net/2022/02/11/GgLPTd9fQvVqpjB.png)

在字符串窗口可以搜索到.datadiv前缀的字符串，这是默认的ollvm混淆字符串的函数 

![%E9%80%9A%E8%BF%87frida%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%205c6fc7e7648b438bb22a21f93c41641f/Untitled%201.png](https://s2.loli.net/2022/02/11/IQAmT5Zut3jwJoU.png)

进入这个函数可以看到字符串都被混淆了

![%E9%80%9A%E8%BF%87frida%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%205c6fc7e7648b438bb22a21f93c41641f/Untitled%202.png](https://s2.loli.net/2022/02/11/XFUN6lWK4Ov1DJx.png)

以异或0xD2为例, 在010Editor的工具中可以恢复这个字符串，十六进制计算中存在异或选项. 无符号字节,十六进制，此时就可以拿到解密以后的结果

ollvm静态的时候没法看到明文字符串，执行以后在.init_array函数里面对字符串进行解密，解密之后内存之中的字符串就是明文字符串了

JNI_OnLoad函数的函数数组偏移是这样的

![%E9%80%9A%E8%BF%87frida%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%205c6fc7e7648b438bb22a21f93c41641f/Untitled%203.png](https://s2.loli.net/2022/02/11/EeolH3gBxqhCJWP.png)

转换指针，可以看到函数，不过函数名被混淆了成了aYcmd

![%E9%80%9A%E8%BF%87frida%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%205c6fc7e7648b438bb22a21f93c41641f/Untitled%204.png](https://s2.loli.net/2022/02/11/oCiv1DerfkcUQ8d.png)

通过frida hook可以打印出真正的函数名，执行下面的脚本，aYcmd解密后还原了明文sign1 

通过这种方式可以还原所有的字符串，只不过工作量会很大

![%E9%80%9A%E8%BF%87frida%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8A%A0%E5%AF%86%205c6fc7e7648b438bb22a21f93c41641f/Untitled%205.png](https://s2.loli.net/2022/02/11/BFlJuG1cPg2Zf6o.png)