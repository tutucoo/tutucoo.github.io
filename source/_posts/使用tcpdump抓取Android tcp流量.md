---
title: 使用tcpdump抓取Android tcp流量
date: 2021-11-04 17:52:23
tags:
---

1、下载tcpdump

[https://www.androidtcpdump.com/android-tcpdump/downloads](https://www.androidtcpdump.com/android-tcpdump/downloads)

2、tcpdump配置

要把tcpdump文件放到/system/bin目录下，设备必须root，否则没有权限

```jsx
adb push tcpdump /data/local/tmp
cat tcpdump /system/bin/tcpdump
```

给tcpdump赋值777权限 

```jsx
chmod 777 /system/bin/tcpdump
```

3、进行tcp通信

![Untitled](https://i.loli.net/2021/11/04/s9alThXIu7mS3QR.png)

4、进行抓包

```jsx
tcpdump -i wlan0 -s 0 -w /sdcard/1.pcap -v
```

ctrl+c可以结束抓包 

5、导出pcap分析

```jsx
adb pull /sdcard/1.pcap
```

![Untitled](https://i.loli.net/2021/11/04/q6zwbuKHOsAQkRv.png)
