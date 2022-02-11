---
title: 算法@分组密码之DES
date: 2022-02-11 15:09:37
tags:
---
# 分组密码之DES

## 概述

DES算法会将明文按64位进行分组，密钥长64位，其中有56位参与DES运算，其余的几位是检验位，分组后的明文会跟56位密钥进行按位替代或交换形成密文

每次加密会对64位明文进行16轮编码，每一轮都会用密钥生成的子密钥进行运算

DES主要的处理过程分为两部分，首先是密钥的生成，然后是明文的处理

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BDES%201e6af285bea940e0a60f26ecade3ae13/Untitled.png](https://s2.loli.net/2022/02/11/mEyAYxR1QSb4Bhu.png)

## DES算法中出现的常量表

在对每一个64位明文分组进行处理的过程中，有大量常量表参与，从而完成对明文的混淆、扩散等处理。主要常量表有：初始置换表、逆初始置换表、扩展置换E表、8个s-box，这些常量都是快速判断DES算法的标志

![Untitled](https://s2.loli.net/2022/02/11/hSwx82KXNmYZLzo.png)

在针对明文分组16轮处理过程中，每一轮都需要一个由原始56位密钥经过编排生成的48位子密钥的参与，这个过程中也出现了一些常量表，主要有：初始置换PC-1表、PC-2表

![Untitled](https://s2.loli.net/2022/02/11/EcxWTZ5LNCmt36J.png)

## 双重DES和三重DES

双重DES就是首先用key1对明文进行加密得到加密字符串，再用key2对加密字符串再进行加密得到最终加密字符串

三重DES是类似的概念，只是加密多加了一轮

## Java中使用DES加密

Java中使用DES算法

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BDES%201e6af285bea940e0a60f26ecade3ae13/Untitled%203.png](https://s2.loli.net/2022/02/11/lKZ1G2kadeSUWy9.png)

iv是初始化向量参数

PKCS5Padding表示分组的填充方式

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BDES%201e6af285bea940e0a60f26ecade3ae13/Untitled%204.png](https://s2.loli.net/2022/02/11/1PwoyYEqrcHkQ4p.png)

## 对DES加密进行hook

下面有些函数需要重载

```jsx
function main(){
	Java.perform(function(){
        Java.use('javax.crypto.Cipher').getInstance.implementation = function (arg0) {
            console.log('javax.crypto.Cipher.getInstance is called!',arg0);
            var result = this.getInstance(arg0);
            return result;
        };
        Java.use('javax.crypto.Cipher').init.implementation = function (arg0,arg1,arg2) {
            console.log('javax.crypto.Cipher.init is called!',arg0,arg1,arg2);
            var mode = arg0;
            var key = arg1;
            var iv = arg2;
            var key_bytes = key.getEncoded();
            var iv_bytes = iv.getIV();
            console.log('javax.crypto.Cipher.init is called!',mode,key_bytes,iv_bytes);
            var result = this.init(arg0,arg1,arg2);
            return result;
        }
    })
}
```

## DES的识别

可以使用findcrypt3进行识别 ，这个插件通过识别常量来判断有没有加密，是哪种加密

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BDES%201e6af285bea940e0a60f26ecade3ae13/Untitled%205.png](https://s2.loli.net/2022/02/11/NUj3snxCvqMDBip.png)

所以我们也可以自己写规则，用来识别加密算法，现在很多用的都不是标准DES的S盒，因此标准DES反而识别不出来，需要自己写规则识别

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BDES%201e6af285bea940e0a60f26ecade3ae13/Untitled%206.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211MIt04G.png)

另外，可以在IDA中搜E_table这样的字符串，然后把字符串的值填到规则里面

![%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E4%B9%8BDES%201e6af285bea940e0a60f26ecade3ae13/Untitled%207.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211fEFP48.png)

当然也可以通过hook api查看DES加密函数有没有被调用