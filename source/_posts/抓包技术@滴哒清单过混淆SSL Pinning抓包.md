---
title: 抓包技术@滴哒清单过混淆SSL Pinning抓包
date: 2021-10-21 14:57:48
cover: https://i.loli.net/2021/10/21/DfkOr7ueYsaH8EV.jpg
---



app版本：5.8.6.1

设备版本：Android8.1

正常对滴哒进行抓包提示Charles不可信，那么就可以猜测使用了SSL Pinning，具体是哪一种pinning要看它的网络框架，对app进行反编译可以看到是okhttp3的网络框架 

![Untitled](https://i.loli.net/2021/10/21/ns4mPQRCcabEy1r.png)

参考DroidSSLUnpinning对Okhttp3的ssl pinner hook代码，可以看到目标hook函数有两个参数，第一个参数是字符串，第二个参数是List，那么只要找这个函数并对它进行hook就可以了，但是这里直接使用这个脚本是无法Hook成功的，原因是app进行了类名函数名混淆，所以先要找到类名和函数名

![Untitled](https://i.loli.net/2021/10/21/RQnvVTk9FcLsrWi.png)

进入objection

```jsx
objection -N -h 192.168.77.161 -p 8888 -g cn.ticktick.task explore
```

搜索File，证书一般会通过File相关的类进行传递，只要找到sslpinning的关键字对其进行hook就可能实现绕过了

```jsx
android hooking search classes File
```

![Untitled](https://i.loli.net/2021/10/21/s4S1FyCMd9IBxzZ.png)

监视这个类的所有方法调用

```jsx
android hooking watch class_method java.io.File.$init --dump-args --dump-backtrace --dump-return
```

看到调用栈中存在CertificatePinner类

![Untitled](https://i.loli.net/2021/10/21/MylakI9N8mDB1qn.png)

加载wallbreaker插件，看下a函数的参数 ，可以看到参数跟上面分析的脚本的参数是一致的，第一个参数都是String，第二个参数是List

![Untitled](https://i.loli.net/2021/10/21/8b49kaZG5fryw1j.png)

看下这个类的实例

![Untitled](https://i.loli.net/2021/10/21/e8u4ajzUgi1Fc5I.png)

dump实例，这样看的更清楚

```jsx
plugin wallbreaker objectdump --fullname 0x22fa
```

![Untitled](https://i.loli.net/2021/10/21/IyP8GDtc7EpakLM.png)

那么只要对这个函数进行hook就可以了

```jsx
function hookCer(){
    Java.perform(function(){
        Java.use("z1.g").a.implementation = function(){
            console.log("z1.g was called !")
            return;
        }
    })
}

setImmediate(hookCer)
```

这样就可以正常抓包了 

![Untitled](https://i.loli.net/2021/10/21/w9ZE853esy4qOIk.png)
