---
title: Frida笔记二： objection
date: 2021-10-09 11:06:12
cover: https://i.loli.net/2021/10/09/mQc3S9s8xNifOJP.jpg
---

# Frida笔记二： objection

## 概述

objection是一个基于Frida开发的工具，可以很方便的hook Java函数和类，并且输出参数，调用栈，返回值。

## **安装objection**

```
 pip install objection
```

![Untitled](https://i.loli.net/2021/10/09/OpWJPNnb49jEsfX.png)

## 常用功能

### **帮助**

```
 help android hooking watch class_method
```

### 注入app

通过wifi的方式启动手机上的app

```
objection -N -h 192.168.1.115 -p 8888 -g com.android.settings explore
```

![Untitled](https://i.loli.net/2021/10/09/vVK9r7XHgIyWNTc.png)

### android heap

execute：执行Java 类方法

search instances：搜索Android堆中的类实例

### android root

尝试禁用root检测 

```php
android root disable
```

尝试模拟root环境 

```php
android root simulate
```

### android intent

启动activity

```php
android intent launch_activity
```

启动服务 

```php
android intent launch_service
```

### android hooking search

搜索类

```php
android hooking search classes
```

搜索方法

```php
android hooking search methods
```

### android hooking list

列出所有activity

```php
android hooking list activities
```

列出所有类

```php
android hooking list classes
```

列出所有类方法

```php
android hooking list class_methods
```

列出所有classloader

```php
	android hooking list class_loaders
```

列出所有监听器

```php
android hooking list receivers
```

列出所有服务 

```php
android hooking list services
```

### android hooking watch

hook类调用

```
android hooking watch class --dump-args --dump-b
acktrace --dump-return
```

hook方法调用

```php
android hooking watch class_method --dump-args -
-dump-backtrace --dump-return
```

### memory dump

dump当前进程的整个内存

```php
memory dump all
```

指定基址进行dump

```php
memory dump from_base
```

### memory search

从指定偏移搜索内存

```php
memory search --offsets-only
```

在内存中搜索字符串

```php
memory search --string
```

### memory write

写内存

```php
memory write --string
```

### memory list

导出模块信息到文件 

```php
memory list exports
```

列出模块信息

```php
memory list modules
```

### job

显示当前任务，比如说android hooking watch一个类

```php
jobs list
```

删除一个任务

```php
jobs kill
```

### objection log

显示objection历史操作信息 

```python
cat .objection/objection.log 
```

### plugin

加载插件

```php
plugin load /root/Wallbreaker
```

可以查看到插件的所有命令

![Untitled](https://i.loli.net/2021/10/09/BPIpo4cROVWH91G.png)

## 插件：Wallbreaker

这个插件在最新的objection中使用bug比较多

classsearch：搜索类名中包含bluetooth的类

```php
plugin wallbreaker classsearch bluetooth
```

classdump：dump类中所有信息

```php
plugin wallbreaker classdump --fullname com.andr
oid.settingslib.bluetooth.BluetoothCallback
```

objectsearch：搜索对象 

```php
plugin wallbreaker objectsearch
```

objectdump：dump对象

```php
plugin wallbreaker objectdump --as-class
plugin wallbreaker objectdump --fullname
```

## 插件：FRIDA-DEXDump

搜索所有dex

```php
plugin dexdump search
```

dump内存中所有的dex

```php
plugin dexdump dump
```

![Untitled](https://i.loli.net/2021/10/09/Wx5M81h3X42uvbk.png)
