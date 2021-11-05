---
title: 抓包技术@Jni层SSL溯源与Hook
date: 2021-11-05 09:57:19
tags:
---

# Jni层SSL溯源与Hook

## 概述

第一种情况：使用Java层SSL库进行通信，hook SSLInputStreamClass.read函数

第二种情况：使用系统lib SSL库(boringssl)进行通信，hook SSL_write函数

第三种情况：自定义Openssl库进行通信，清除掉符号的情况hook read函数，没有清除符号还是hook SSL_write函数，自定义openssl库一般还是会调用这个函数

## jni层SSL溯源

Java层最后调用的函数是SSL_write，Android源码中找到SSL_write函数 

![Untitled](https://i.loli.net/2021/11/05/dJnGRPb6MA9wkFS.png)

找到所在的cpp文件，找到代码实现

![Untitled](https://i.loli.net/2021/11/05/NqwAkJxiUr7b6Oa.png)

NativeCrypto_SSL_write函数实现

![Untitled](https://i.loli.net/2021/11/05/tyAZLVGNjl53KYW.png)

接着在内部调用sslWrite函数  

![Untitled](https://i.loli.net/2021/11/05/WLQZlFrTcsi3uSw.png)

然后调用SSL_write函数 

![Untitled](https://i.loli.net/2021/11/05/PqzrD86LpvFTWYm.png)

全局搜索，在boringssl库中找到SSL_write函数 

![Untitled](https://i.loli.net/2021/11/05/LKvBihts5roApMO.png)

找到它的实现

![Untitled](https://i.loli.net/2021/11/05/3qvUBzHZG7Rhokj.png)

看下SSL_write的实现

![Untitled](https://i.loli.net/2021/11/05/tmlDBRxGrsPA59I.png)

内部调用ssl→method→write_app_data

![Untitled](https://i.loli.net/2021/11/05/VteQ5DMsUpgnSc6.png)

这个ssl里面有method常量

![Untitled](https://i.loli.net/2021/11/05/WSyUcHXTAGZxw9g.png)

找到ssl_protocol_method_st

![Untitled](https://i.loli.net/2021/11/05/vX2YzkmhcO9bquB.png)

内部存在一个write_app_data函数指针

![Untitled](https://i.loli.net/2021/11/05/hDAYaLfrxZkGB3u.png)

根据不同的协议会调用不同的实现，这里会调用ssl3_write_app_data

![Untitled](https://i.loli.net/2021/11/05/IoCtLdg8iuA1pzZ.png)

进入ssl3_writre_app_data方法

![Untitled](https://i.loli.net/2021/11/05/gnHjUyZrkoa1NxQ.png)

看到do_ssl3_write方法，执行完这个方法后明文就会进行加密

![Untitled](https://i.loli.net/2021/11/05/mQ3XktfPARpas18.png)

接着进入到ssl3_write_pending函数 

![Untitled](https://i.loli.net/2021/11/05/PfdDlY6W9eTxgAH.png)

然后调用ssl_write_buffer_flush

![Untitled](https://i.loli.net/2021/11/05/Gnrpcx5KEid6RkJ.png)

然后会进入tls_write_buffer_flush

![Untitled](https://i.loli.net/2021/11/05/3hRPWaDy2C8wszF.png)

然后进入到BIO_write

![Untitled](https://i.loli.net/2021/11/05/Xs4rSJUaeWAq5gb.png)

内部调用了bio_io

![Untitled](https://i.loli.net/2021/11/05/lIXtex5gBpHmucZ.png)

进入bio_io

![Untitled](https://i.loli.net/2021/11/05/JU8DKXpgt9TZjmQ.png)

找到boringssl源码，最终会调用sock_read和sock_write，内部又调用了read和write函数 ，注意这里细节，ssl底层发送没有使用send和recv，而是使用了write和read

![Untitled](https://i.loli.net/2021/11/05/e5RyJHIsS617U9C.png)

## JNI SSL Hook

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

function printNativeStack(context, name) {
    //Debug.
    var array = Thread.backtrace(context, Backtracer.ACCURATE);
    var first = DebugSymbol.fromAddress(array[0]);
    if (first.toString().indexOf('libopenjdk.so!NET_Send') < 0) {
        var trace = Thread.backtrace(context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join("\n");
        LogPrint("-----------start:" + name + "--------------");
        LogPrint(trace);
        LogPrint("-----------end:" + name + "--------------");
    }

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

function getsocketdetail(fd) {
    var result = "";
    var type = Socket.type(fd);
    if (type != null) {
        result = result + "type:" + type;
        var peer = Socket.peerAddress(fd);
        var local = Socket.localAddress(fd);
        result = result + ",address:" + JSON.stringify(peer) + ",local:" + JSON.stringify(local);
    } else {
        result = "unknown";
    }
    return result;

}

function getip(ip_ptr) {
    var result = ptr(ip_ptr).readU8() + "." + ptr(ip_ptr.add(1)).readU8() + "." + ptr(ip_ptr.add(2)).readU8() + "." + ptr(ip_ptr.add(3)).readU8()
    return result;
}

function hooklibc() {
    var libcmodule = Process.getModuleByName("libc.so");
    var read_addr = libcmodule.getExportByName("read");
    var write_addr = libcmodule.getExportByName("write");
    console.log(read_addr + "---" + write_addr);
    Interceptor.attach(read_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];

            this.socketinfo = getsocketdetail(this.arg0.toInt32());
            LogPrint("go into libc.so->read_addr" + "---" + this.socketinfo);
            this.flag = false;
            if (this.socketinfo.indexOf("tcp") >= 0) {
                this.flag = true;
            }
            if (this.flag) {
                printNativeStack(this.context, Process.getCurrentThreadId() + "read");
            }

        }, onLeave(retval) {

            if (this.flag) {
                var size = retval.toInt32();
                if (size > 0) {
                    console.log(Process.getCurrentThreadId() + "---libc.so->read:" + hexdump(this.arg1, {
                        length: size
                    }));
                }
            }

            LogPrint("leave libc.so->read");
        }
    });
    Interceptor.attach(write_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];

            this.socketinfo = getsocketdetail(this.arg0.toInt32());
            LogPrint("go into libc.so->write" + "---" + this.socketinfo);
            this.flag = false;
            if (this.socketinfo.indexOf("tcp") >= 0) {
                this.flag = true;
            }
            if (this.flag) {
                printNativeStack(this.context, Process.getCurrentThreadId() + "write");
            }

        }, onLeave(retval) {
            if (this.flag) {
                var size = ptr(this.arg2).toInt32();
                if (size > 0) {
                    console.log(Process.getCurrentThreadId() + "---libc.so->write:" + hexdump(this.arg1, {
                        length: size
                    }));
                }
            }

            LogPrint("leave libc.so->write");
        }
    });
}

function hookssl() {
    var libcmodule = Process.getModuleByName("libssl.so");
		//boringssl库中的函数，此时的流量是明文 
    var read_addr = libcmodule.getExportByName("SSL_read");
    var write_addr = libcmodule.getExportByName("SSL_write");
    Interceptor.attach(read_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];
            LogPrint("go into libssl.so->SSL_read");

            printNativeStack(this.context, Process.getCurrentThreadId() + "SSL_read");

        }, onLeave(retval) {

            var size = retval.toInt32();
            if (size > 0) {
                console.log(Process.getCurrentThreadId() + "---libssl.so->SSL_read:" + hexdump(this.arg1, {
                    length: size
                }));
            }
            LogPrint("leave libssl.so->SSL_read");
        }
    });
    Interceptor.attach(write_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];
            LogPrint("go into libssl.so->SSL_write");

            printNativeStack(this.context, Process.getCurrentThreadId() + "SSL_write");

        }, onLeave(retval) {
            var size = ptr(this.arg2).toInt32();
            if (size > 0) {
                console.log(Process.getCurrentThreadId() + "---libssl.so->SSL_write:" + hexdump(this.arg1, {
                    length: size
                }));
            }

            LogPrint("leave libssl.so->SSL_write");
        }
    });
}

function main() {
		//可以获取加密后的流量
    hooklibc();
		//可以获取到明文 
    //hookssl();
}

setImmediate(main);
```

调用hooklibc函数可以看到流量都是加密的，hookssl函数中对libssl.so中的SSL_write和SSL_read进行了hook，这两个函数是boringssl库的函数，可以获取到明文 

![Untitled](https://i.loli.net/2021/11/05/NXBudnM41fRpCig.png)

调用hookssl函数可以获取到明文  

![Untitled](https://i.loli.net/2021/11/05/fQyvwkr4psibq2Y.png)

用c编写openssl时，需要通过SSL_set_fd函数将socket跟SSL进行绑定，参数一个为SSL *ssl,另一个为int fd，如果我们要获取通信的ip的话，我们需要直接对fd进行获取，然后再使用我们上一节的解析函数，使用frida的api进行获取ip和端口

发现在libssl中也有这个函数可以直接获取fd的，SSL_get_fd()，方法是可以直接返回的fd的，所以我们需要去主动调用它

```jsx
var libcmodule = Process.getModuleByName("libssl.so");
var read_addr = libcmodule.getExportByName("SSL_read");
var write_addr = libcmodule.getExportByName("SSL_write");
var SSL_get_rfd_ptr=libcmodule.getExportByName("SSL_get_rfd");
var SSL_get_rfd=new NativeFunction(SSL_get_rfd_ptr,'int',['pointer']);
```

![Jni%E5%B1%82SSL%E6%BA%AF%E6%BA%90%E4%B8%8EHook%2024f8dd39937a490b95ecf5d2d1ced5c0/Untitled%2024.png](https://i.loli.net/2021/11/05/17aphuEcxvCBfFY.png)