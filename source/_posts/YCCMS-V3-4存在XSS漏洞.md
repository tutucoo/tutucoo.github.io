---
title: YCCMS V3 4存在XSS漏洞
date: 2021-04-17 15:50:57
cover: https://i.loli.net/2021/04/17/CS146wha5HeK72p.png
---


## 环境准备

CMS下载地址: [http://ahdx.down.chinaz.com/202003/yccms_v3.4.rar](http://ahdx.down.chinaz.com/202003/yccms_v3.4.rar)

该cms根目录下有个安装文档，相关设置都在里面

有个坑，mysql需要预先创建一个my_admin数据库，否则导入会失败

## 漏洞复现

XSS payload 

```python
http:xxx.xxx/admin/?a=html&art=<sCrIpT>alert(1)<%2FsCrIpT>&m=arts
```

![YCCMS%20V3%204%E5%AD%98%E5%9C%A8XSS%E6%BC%8F%E6%B4%9E%20f45f98e06c1a4aca8f78afffaca26d47/Untitled.png](https://i.loli.net/2021/04/17/IX9kiWuKaR7UhOt.png)

## 漏洞原理

点击“开始生成”，界面上更新文字，“内容生成完毕！共6条”，猜测这里执行了js代码

![YCCMS%20V3%204%E5%AD%98%E5%9C%A8XSS%E6%BC%8F%E6%B4%9E%20f45f98e06c1a4aca8f78afffaca26d47/Untitled%201.png](https://i.loli.net/2021/04/17/iqPEA6yZOLR5H1S.png)

抓包，a参数对应根目录下的控制器，m参数表示方法，art参数是生成文章的数量 

![YCCMS%20V3%204%E5%AD%98%E5%9C%A8XSS%E6%BC%8F%E6%B4%9E%20f45f98e06c1a4aca8f78afffaca26d47/Untitled%202.png](https://i.loli.net/2021/04/17/UDIZdazr6fViFO1.png)

art的参数赋值为<script>alert(1)</script>直接弹窗了

我们再看下源码，找到html控制器

![YCCMS%20V3%204%E5%AD%98%E5%9C%A8XSS%E6%BC%8F%E6%B4%9E%20f45f98e06c1a4aca8f78afffaca26d47/Untitled%203.png](https://i.loli.net/2021/04/17/Qk1eJbdn38w7Phm.png)

找到arts函数，可以看到生成完成后会使用document.body.innerHTML写入到网页中，从而执行了js代码，

![YCCMS%20V3%204%E5%AD%98%E5%9C%A8XSS%E6%BC%8F%E6%B4%9E%20f45f98e06c1a4aca8f78afffaca26d47/Untitled%204.png](https://i.loli.net/2021/04/17/6xSJHE9O1rjXw4a.png)

前端插入xss后的源码

![YCCMS%20V3%204%E5%AD%98%E5%9C%A8XSS%E6%BC%8F%E6%B4%9E%20f45f98e06c1a4aca8f78afffaca26d47/Untitled%205.png](https://i.loli.net/2021/04/17/eHMOXcfTE29F8Qs.png)

继续看“内容生成完毕”所在的位置，可以看到在art被赋值为一段js代码后，最终得到了执行

![YCCMS%20V3%204%E5%AD%98%E5%9C%A8XSS%E6%BC%8F%E6%B4%9E%20f45f98e06c1a4aca8f78afffaca26d47/Untitled%206.png](https://i.loli.net/2021/04/17/kP7bG25BvYgiV6H.png)