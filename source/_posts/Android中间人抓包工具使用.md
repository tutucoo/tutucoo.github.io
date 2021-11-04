---
title: Android中间人抓包工具使用
date: 2021-11-04 17:41:52
tags:
---

# Android中间人抓包工具使用

## **使用Charles进行抓包**

### charles破解

官网下载charles，然后进入破解网站生成序列号直接注册成功

[https://www.zzzmode.com/mytools/charles/](https://www.zzzmode.com/mytools/charles/)

1、手机设置代理  

wifi设置代理，端口8888

2、charles设置

Proxy-->SSL Proxying Setting，Enable SSL Proxying打上勾，ip和端口全部*

![Untitled](https://i.loli.net/2021/11/04/EaJ5v3l4NoP8krO.png)

3、手机安装charles证书

手机访问 chls.pro/ssl 下载并安装证书

### 开启charles外部代理

External Proxy Settings-->Use external proxy servers打上勾

![Untitled](https://i.loli.net/2021/11/04/xRlQYLz5ViyA1Bq.png)

### **配合Postern拦截socks5流量**

1、Charles端设置

开启socks代理 

![Untitled](https://i.loli.net/2021/11/04/Qhj9cEoZ6M1HgPG.png)

通过socks可以抓取http额外端口的数据

![Untitled](https://i.loli.net/2021/11/04/sWlQvMqd96yTijG.png)

2、postern端配置

配置代理

```jsx
服务器地址：填Charles主机ip

服务器端口：Charles默认8889

代理类型：SOCKS5
```

配置规则

```jsx
匹配类型：匹配所有地址
动作：通过代理连接
代理/代理组：上一步配置的代理 
```

3、开启抓包

点击开启VPN

## 使用burp抓包

抓包的一般过程跟charles类似就不多说了，需要注意的是burp抓包android https时需要提前对证书进行转换

burp提供的在线下载是der格式的文件，需要先导出进行转换

```jsx
openssl x509 -inform DER -in test.der -out burp.pem
```

push到sdcard中

```jsx
adb push burp.pem /sdcard
```

进入手机设置-加密与凭据-从存储设备安装，点击刚刚push到手机上面的burp.pem进行安装就可以了 

### 开启外部代理

外部代理设置为梯子的端口，这样经过burp的流量就可以访问外网了

![Untitled](https://i.loli.net/2021/11/04/XD2y9FHOBS5C8rL.png)

## 证书移动到根目录

默认安装的证书在用户目录，可以通过安装magisk插件Move Certificates自动把证书移到root目录 

也可以手动移动 

```
cd /data/misc/user/0/cacerts-added
mount -o remount,rw /
cp * /etc/security/cacerts/
mount -o remount,ro /
```

