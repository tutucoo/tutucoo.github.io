---
title: 算法@RC4算法
date: 2022-02-11 15:00:37
tags:
---
# RC4算法

## 原理

RC4算法主要有两个算法构成，一个是初始化算法（KSA），还有一个是伪随机子密码生成算法（PRGA）

![Untitled](https://s2.loli.net/2022/02/11/sg2PYmUJLtMQfkA.png)

KSA算法部分，参数1是一个长度为256的char数组，参数2是密钥，可以任意定义，参数3是密钥的长度，密钥的主要功能是将s-box打乱

![Untitled](https://s2.loli.net/2022/02/11/hYlXEuRKs2q8cez.png)

PRGA算法，参数1是被打乱的s-box，参数2是需要加密的数据

![Untitled](https://s2.loli.net/2022/02/11/Z1xzkGXUrbIfYn5.png)

## RC4加密算法的识别

1、RC4算法加密的字符串有一个特点，明文和密文长度相等

2、逆向算法，找到KSA算法，有两轮非常明显的长度为256的循环体