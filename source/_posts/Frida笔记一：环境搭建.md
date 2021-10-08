---
title: Frida笔记一：环境搭建
date: 2021-10-08 15:59:12
cover: https://i.loli.net/2021/10/08/eaYgl3RL2izHDKJ.jpg
---

# Frida笔记一：环境搭建

## **安装python3**

frida需要python3的支持

```
 pyenv install 3.7.5 -v
```

## **安装frida**

安装frida(安装frida-tools会自动安装最新版frida，所以也要指定frida-tools的版本)

```
 pip install frida==12.4.8
```

安装frida tool(命令行工具)

```
 pip install frida-tools==1.3.2
```

查看frida版本

```
 frida --version
```

## 运行**frida server**

在此页面[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)

下载frida-server-12.4.8-android-arm64.xz，frida-server的版本与前面安装的frida版本要一致

解压frida-server-12.4.8-android-arm64.xz释放出frida-server-12.4.8-android-arm64

```
 $ adb push frida-server-12.4.8-android-arm64 /data/local/tmp/fs1248
 $ adb shell chmod +x /data/local/tmp/fs1248
```

需要注意的是，在启动frida server之前要确保手机selinux是关闭的状态，可以通过指令进行设置

```
 set enforce 0
```

此外还要关闭Magisk的Magisk Hide功能

启动服务器

```
 /data/local/tmp/fs1248 -l 0.0.0.0:8888 
```

## frida脚本开发环境

进行环境安装 

```php
git clone git://github.com/oleavr/frida-agent-example.git
cd frida-agent-example/
npm install
```

用vscode打开firda-agent-example文件，然后在agent下编写脚本就有提示了 

![Untitled](https://i.loli.net/2021/10/08/N5TX2QLhjItrYsc.png)

进入frida-agent-example文件夹执行下面的命令会监控修改并自动生成js文件 

```php
npm run watch
```

执行脚本 

```php
frida -f com.android.settings -H 192.168.1.115:8888 -l test.js --no-pause
```

![Untitled](https://i.loli.net/2021/10/08/3wSUdzC9VqsNKpA.png)
