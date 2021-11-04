---
title: Java层Socket溯源与Hook
date: 2021-11-04 17:54:57
tags:
---

# J**ava层socket溯源**

可以通过AS动态调试进行溯源

```
java.net.Socket类构造函数: new Socket(ip,port);->Socket(InetAddress[] addresses,int port,SocketAddress localAddr,boolean stream)>impl->java.net.SocksSocketImpl->

建立连接：connect(SocketAddress endpoint)

接收数据：java.net.SocketInputStream.read(byte[])->read(b,0,b.length)->read(b,off,length,impl.getTimeout())->socketRead(fd, b, off, length, timeout);(native) 

发送数据: java.net.SocketOutputStream.write(byte[])->socketWrite(b,0,b.length)->socketWrite0(fd,b,off,len)
```

接收数据的read函数所在的类是一个虚类，那么就有一个实现的类，可以动态调试时看这个对象的shadow$_klass的值，这个值就是它的实现类，这里的值是java.net.SocketInputStream，再通过查看源码找到这个类的具体实现

可以看到接收数据调用read函数后，最终会调用socketRead0函数，这个函数是native层的函数，只要对这个函数进行hook就可以拦截所有的tcp流量

![Untitled](https://i.loli.net/2021/11/04/V79YrByHfgeLUMt.png)

有些情况是通过udp进行通信的，比如说http/2就是通过发送udp请求进行通信的

发送数据，源码函数位于java.net.DatagramSocket.send

![Untitled](https://i.loli.net/2021/11/04/YMtnd2KG8bToE1u.png)

send内部调用了getImpl.send函数

![Untitled](https://i.loli.net/2021/11/04/d8tMnQOh24AEL6s.png)

getImpl函数的作用是获取实例，通过这个实例来执行send

getImpl内部调用了createImpl函数

![Untitled](https://i.loli.net/2021/11/04/FzEwICj6gd7QUp8.png)

通过动态调试知道获取的实例是PlainDatagramSocketImpl

![Untitled](https://i.loli.net/2021/11/04/ozyZFJVcAKRgj8f.png)

找到PlainDatagramSocketImpl.send的实现，内部调用了IoBridge.sendto

![Untitled](https://i.loli.net/2021/11/04/p98cVdLWPMIOqlv.png)

再进入内部是Libcore.os.sendto

![Untitled](https://i.loli.net/2021/11/04/YlympP8UOCzB7Qb.png)

这里直接跳转sendto，找不到正确的位置，需要先找到Libcore类

Libcore类的内部有个os变量，属于BlockGuardOs类

![Untitled](https://i.loli.net/2021/11/04/KFmoXaQTVRfHvn7.png)

进入BlockGuardOs类找到sendto

![Untitled](https://i.loli.net/2021/11/04/S1lCjbFiUkqm8tL.png)

这里可以看到通过os调用的sendto，os是在BlockGuardOs初始化时传入的，是在父类中进行处理的

![Untitled](https://i.loli.net/2021/11/04/TBQKIeZ5D4iwqCX.png)

os属于Os类

![Untitled](https://i.loli.net/2021/11/04/1XVQZgOWenpPJLo.png)

Os类有sendto接口

![Untitled](https://i.loli.net/2021/11/04/O3tQYZiA1d8qaJB.png)

通过搜索sendto，找到Linux.java

![Untitled](https://i.loli.net/2021/11/04/6xDSikNloyWbQGu.png)

这样就找到了这个函数的实现，最终调用了native层的setntoBytes

![Untitled](https://i.loli.net/2021/11/04/9gJW4NTm5pvVM3O.png)

接收数据，跟发送数据类似，找到DatagramSocket的receive函数，内部调用了getImpl

![Untitled](https://i.loli.net/2021/11/04/uhMNUxABGc5X2yz.png)

通过动态调试获取到PlainDatagramSocketImp，找到receive函数，内部调用了doRecv

![Untitled](https://i.loli.net/2021/11/04/aubsVve96oqWHkP.png)

doRecv内部调用了IoBridge.recvfrom

![Untitled](https://i.loli.net/2021/11/04/iv6O4jC1Bhl9JYe.png)

然后找到Libcore.os.recvfrom

![Untitled](https://i.loli.net/2021/11/04/AXuhE8lOmStMDdC.png)

直接搜索recvfrom可以看到在Linux.java中有实现，找到最终的函数recvfromBytes

![Untitled](https://i.loli.net/2021/11/04/pNMZfoE5hV7QqBl.png)

# hook Java层tcp socket接口

运行tcp服务器和客户端，方便进行测试

服务端

```jsx
# coding=utf-8
import threading
import socket

socket_list = []

s = socket.socket()
s.bind(('0.0.0.0', 9999))
s.listen()
def read_from_client(s):
    try:
        return s.recv(1024).decode('utf-8')
    except:
        socket_list.remove(s)
def server_target(s):
    try:
        while True:
            content = read_from_client(s)
            if content is None:
                break
            length=len(content)
            if length>0:
                print("receive:"+content)
                response=content+" from server"
                print("send:"+response)
                s.send(response.encode('utf-8'))
    except IOError as e:
        print(e.strerror)

while True:
    c, addr = s.accept() 
    socket_list.append(c)
    threading.Thread(target=server_target, args=(c,)).start()
```

客户端

```jsx
public class TcpClient {

    public static String ip = "192.168.18.244";
    public static int port = 9999;
    public static boolean connected = false;
    public static Socket socket = null;
    public static OutputStream outputstream = null;
    public static InputStream inputStream = null;
    public static long lastheartresponse = 0;

    public static void start() {
        servicethread();
    }

    public static Long getTimestamp() {
        Date date = new Date();
        if (null == date) {
            return (long) 0;
        }
        String timestamp = String.valueOf(date.getTime());
        return Long.valueOf(timestamp);
    }

    public static void close() {
        try {
            if (socket != null) {
                socket.close();
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
        inputStream = null;
        outputstream = null;
        connected = false;
    }

    public static void sendmsg(final String msg) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                if (connected == false) {
                } else {
                    try {
                        if (outputstream != null) {
                            String crypt = msg;
                            outputstream.write(crypt.getBytes("utf-8"));
                            outputstream.flush();
                        }

                    } catch (IOException e) {
                        close();
                        e.printStackTrace();
                    }
                }
            }
        }).start();

    }

    public static void heartthread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    long currenttime = getTimestamp();
                    if (lastheartresponse != 0) {
                        long offset = currenttime - lastheartresponse;
                        int seconds = (int) (offset / 1000);
                        if (seconds > 10) {
                            close();
                        }
                    }
                    try {
                        Thread.currentThread().sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }

            }
        }).start();
    }

    public static void receivethread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                int arraysize = 1024;
                byte[] content = new byte[arraysize];
                while (true) {
                    if (inputStream != null) {
                        try {
                            int count = inputStream.read(content);
                            if (count > 0 && count < arraysize) {
                                byte[] tmparray = new byte[count];
                                System.arraycopy(content, 0, tmparray, 0, count);
                                String str = new String(tmparray, "utf-8");

                            }
                        } catch (IOException e) {
                            e.printStackTrace();
                            close();
                            break;
                        }
                    } else {
                        close();
                        break;
                    }
                }
            }
        }).start();
    }

    public static void servicethread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                heartthread();
                while (true) {
                    if (connected == false) {
                        try {
                            socket = new Socket(ip, port);
                            socket.setSoTimeout(10*1000);
                            connected = true;
                            outputstream = socket.getOutputStream();
                            inputStream = socket.getInputStream();
                            receivethread();
                        } catch (IOException e) {
                            e.printStackTrace();
                            connected = false;
                            socket = null;
                            outputstream = null;
                            inputStream = null;
                        }
                    }
                    if (outputstream != null) {
                        try {
                            JSONObject object = new JSONObject();
                            object.put("msgtype", "tutucoo");
                            sendmsg(object.toString());
                        } catch (JSONException e) {
                            e.printStackTrace();
                        }
                    }
                    try {
                        Thread.currentThread().sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }
            }
        }).start();
    }
}
```

在AndroidManifest.xml文件中添加权限 

```python
<uses-permission android:name="android.permission.INTERNET"/>
```

在MainActivity中启动客户端

```python
TcpClient.start();
```

frida脚本如下：

```jsx
function LogPrint(log) {
    var theDate = new Date();
    var hour = theDate.getHours();
    var minute = theDate.getMinutes();
    var second = theDate.getSeconds();
    var mSecond = theDate.getMilliseconds();

    hour < 10 ? hour = "0" + hour : hour;
    minute < 10 ? minute = "0" + minute : minute;
    second < 10 ? second = "0" + second : second;
    mSecond < 10 ? mSecond = "00" + mSecond : mSecond < 100 ? mSecond = "0" + mSecond : mSecond;
    var time = hour + ":" + minute + ":" + second + ":" + mSecond;
    var threadid = Process.getCurrentThreadId();
    console.log("[" + time + "]" + "->threadid:" + threadid + "--" + log);

}

function printJavaStack(name) {
    Java.perform(function () {
        var Exception = Java.use("java.lang.Exception");
        var ins = Exception.$new("Exception");
        var straces = ins.getStackTrace();
        if (straces != undefined && straces != null) {
            var strace = straces.toString();
            var replaceStr = strace.replace(/,/g, " \n ");
            LogPrint("=============================" + name + " Stack strat=======================");
            LogPrint(replaceStr);
            LogPrint("=============================" + name + " Stack end======================= \n ");
            Exception.$dispose();
        }
    });
}

function isprintable(value) {
    if (value >= 32 && value <= 126) {
        return true;
    }
    return false;
}

function hooktcp() {
    Java.perform(function () {
        var SocketClass = Java.use('java.net.Socket');
				//构造函数的参数分别是ip和port
        SocketClass.$init.overload('java.lang.String', 'int').implementation = function (arg0, arg1) {
            console.log("[" + Process.getCurrentThreadId() + "]new Socket connection:" + arg0 + ",port:" + arg1);
            printJavaStack('tcp connect...')
            return this.$init(arg0, arg1);
        }
        var SocketInputStreamClass = Java.use('java.net.SocketInputStream');
        SocketInputStreamClass.socketRead0.implementation = function (arg0, arg1, arg2, arg3, arg4) {
            var size = this.socketRead0(arg0, arg1, arg2, arg3, arg4);
            var bytearray = Java.array('byte', arg1);
            var content = '';
            for (var i = 0; i < size; i++) {
                if (isprintable(bytearray[i])) {
                    content = content + String.fromCharCode(bytearray[i]);
                }
            }
            var socketimpl = this.impl.value;
            var address = socketimpl.address.value;
            var port = socketimpl.port.value;
						//ip、port、内容 
            console.log("\naddress:" + address + ",port" + port + "\n" + JSON.stringify(this.socket.value) + "\n[" + Process.getCurrentThreadId() + "]receive:" + content);
            printJavaStack('socketRead0')
            return size;
        }
        var SocketOutPutStreamClass = Java.use('java.net.SocketOutputStream');
        SocketOutPutStreamClass.socketWrite0.implementation = function (arg0, arg1, arg2, arg3) {
            var result = this.socketWrite0(arg0, arg1, arg2, arg3);
            //console.log("[" + Process.getCurrentThreadId() + "]socketWrite0:len:" + arg3 + "--content:" + JSON.stringify(arg1));
            var bytearray = Java.array('byte', arg1);
            var content = '';
            for (var i = 0; i < arg3; i++) {

                if (isprintable(bytearray[i])) {
                    content = content + String.fromCharCode(bytearray[i]);
                }
            }
            var socketimpl = this.impl.value;
            var address = socketimpl.address.value;
            var port = socketimpl.port.value;
						//ip、port、内容 
            console.log("send address:" + address + ",port" + port + "[" + Process.getCurrentThreadId() + "]send:" + content);
            console.log("\n" + JSON.stringify(this.socket.value) + "\n[" + Process.getCurrentThreadId() + "]send:" + content);
            printJavaStack('socketWrite0')
            return result;
        }
    })
}

function main() {
    hooktcp();
}

setImmediate(main)
```

打印出了ip、port、传输内容等信息

![Untitled](https://i.loli.net/2021/11/04/E8vLb3cz7k1VDq5.png)

# hook Java层udp socket接口

udp服务端

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import socket
import threading
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(("0.0.0.0", 8888))
print("UDP bound on port 8888...")

while True:
    data, addr = s.recvfrom(1024)
    print("Receive from %s:%s" %(data,addr))
    if data == b"exit":
        s.sendto(b"Good bye!\n", addr)
        continue
    response=str(data)+" received from udpserver"
    s.sendto(response.encode("utf-8"), addr)
```

udp客户端

```java
package com.example.myapplication;

import android.util.Log;

import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.SocketException;

public class UdpClient {
    public static final int DEST_PORT = 8888;
    public static final String DEST_IP = "192.168.2.104";
    public static final int DATA_LEN = 4096;
    public static byte[] inBuff = new byte[DATA_LEN];
    public static DatagramSocket socket;

    static {
        try {
            socket = new DatagramSocket();
            receivethread();
        } catch (SocketException e) {
            e.printStackTrace();
        }
    }

    public static DatagramPacket inPacket = new DatagramPacket(inBuff, inBuff.length);
    public static DatagramPacket outPacket = null;

    public static void udpsend(String content) {
        try {
            outPacket = new DatagramPacket(new byte[0], 0, InetAddress.getByName(DEST_IP), DEST_PORT);
            byte[] buff = content.getBytes();
            outPacket.setData(buff);
            socket.send(outPacket);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void receivethread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        socket.receive(inPacket);
                        Log.i("udpreceive", new String(inBuff, 0, inPacket.getLength()));
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }

    public static void start() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        Thread.currentThread().sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    udpsend("i am from udpclient!");
                }
            }
        }).start();

    }
}
```

AndroidManifest.xml中添加网络权限 

```java
<uses-permission android:name="android.permission.INTERNET"/>
```

MainActivity中调用 

```java
UdpClient.start();
```

hook脚本

```jsx
function LogPrint(log) {
    var theDate = new Date();
    var hour = theDate.getHours();
    var minute = theDate.getMinutes();
    var second = theDate.getSeconds();
    var mSecond = theDate.getMilliseconds();

    hour < 10 ? hour = "0" + hour : hour;
    minute < 10 ? minute = "0" + minute : minute;
    second < 10 ? second = "0" + second : second;
    mSecond < 10 ? mSecond = "00" + mSecond : mSecond < 100 ? mSecond = "0" + mSecond : mSecond;
    var time = hour + ":" + minute + ":" + second + ":" + mSecond;
    var threadid = Process.getCurrentThreadId();
    console.log("[" + time + "]" + "->threadid:" + threadid + "--" + log);

}

function printJavaStack(name) {
    Java.perform(function () {
        var Exception = Java.use("java.lang.Exception");
        var ins = Exception.$new("Exception");
        var straces = ins.getStackTrace();
        if (straces != undefined && straces != null) {
            var strace = straces.toString();
            var replaceStr = strace.replace(/,/g, " \n ");
            LogPrint("=============================" + name + " Stack strat=======================");
            LogPrint(replaceStr);
            LogPrint("=============================" + name + " Stack end======================= \n ");
            Exception.$dispose();
        }
    });
}

function isprintable(value) {
    if (value >= 32 && value <= 126) {
        return true;
    }
    return false;
}

function hookudp() {
    Java.perform(function () {
        var LinuxClass = Java.use('libcore.io.Linux');
        //private native int recvfromBytes(FileDescriptor fd, Object buffer, int byteOffset, int byteCount, int flags, InetSocketAddress srcAddress) throws ErrnoException, SocketException;
        LinuxClass.recvfromBytes.implementation = function (arg0, arg1, arg2, arg3, arg4, arg5) {
            var size = this.recvfromBytes(arg0, arg1, arg2, arg3, arg4, arg5);
            var bytearray = Java.array('byte', arg1);
            var content = "";
            for (var i = 0; i < size; i++) {

                if (isprintable(bytearray[i])) {
                    content = content + String.fromCharCode(bytearray[i]);
                }

            }
            console.log("address:" + arg5 + " [" + Process.getCurrentThreadId() + "]recvfromBytes:size:" + size + ",content:" + JSON.stringify(arg1) + "---content," + content);
            printJavaStack('recvfromBytes');
            return size;
        }
        //private native int sendtoBytes(FileDescriptor fd, Object buffer, int byteOffset, int byteCount, int flags, InetAddress inetAddress, int port) throws ErrnoException, SocketException;
        // private native int sendtoBytes(FileDescriptor fd, Object buffer, int byteOffset, int byteCount, int flags, SocketAddress address) throws ErrnoException, SocketException;
        LinuxClass.sendtoBytes.overload('java.io.FileDescriptor', 'java.lang.Object', 'int', 'int', 'int', 'java.net.InetAddress', 'int').implementation = function (arg0, arg1, arg2, arg3, arg4, arg5, arg6) {
            var size = this.sendtoBytes(arg0, arg1, arg2, arg3, arg4, arg5, arg6);
            var bytearray = Java.array('byte', arg1);
            var content = "";
            for (var i = 0; i < size; i++) {
                if (isprintable(bytearray[i])) {
                    content = content + String.fromCharCode(bytearray[i]);
                }

            }
            console.log("address:" + arg5 + ",port" + arg6 + " [" + Process.getCurrentThreadId() + "]LinuxClass11.sendtoBytes:len:" + size + "--content:" + JSON.stringify(arg1) + "--content:" + content);
            printJavaStack('LinuxClass11.sendtoBytes')
            return size;
        }
				//sendtoBytes有两个实现，因此都要进行hook
        LinuxClass.sendtoBytes.overload('java.io.FileDescriptor', 'java.lang.Object', 'int', 'int', 'int', 'java.net.SocketAddress').implementation = function (arg0, arg1, arg2, arg3, arg4, arg5) {
            var size = this.sendtoBytes(arg0, arg1, arg2, arg3, arg4, arg5);
            var bytearray = Java.array('byte', arg1);
            var content = "";
            for (var i = 0; i < size; i++) {
                if (isprintable(bytearray[i])) {
                    content = content + String.fromCharCode(bytearray[i]);
                }
            }
            console.log("address:" + arg5 + " [" + Process.getCurrentThreadId() + "]LinuxClass22.sendtoBytes:len:" + size + "--content:" + JSON.stringify(arg1) + ",content:" + content);
            printJavaStack('LinuxClass22.sendtoBytes')
            return size;
        }

    })

}

function main() {
    hookudp();
}

setImmediate(main)
```

![Untitled](https://i.loli.net/2021/11/04/6WdQKn149VRxC3E.png)
