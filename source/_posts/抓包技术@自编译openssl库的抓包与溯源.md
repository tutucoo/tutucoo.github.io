---
title: 抓包技术@自编译openssl库的抓包与溯源
date: 2021-11-05 09:57:19
tags:
---

# 自编译openssl库的抓包与溯源

对于没有抹掉符号的，直接hook SSL_read和SSL_write，SSL_read和SSL_write函数是有特征的，它会调用j_ERR_put_error函数

![Untitled](https://i.loli.net/2021/11/05/sXQkytV9zfEwYoR.png)

符号未被抹去时，遍历所有模块，找到SSL_write进行hook，hook代码如下

```java
Process.enumerateModules().forEach(function (module) {
        module.enumerateExports().forEach(function (symbol) {
            var name = symbol.name;
            if (name == 'SSL_read') {
                LogPrint(JSON.stringify(module) + JSON.stringify(symbol));
            }
            if (name == 'SSL_write') {
                LogPrint(JSON.stringify(module) + JSON.stringify(symbol));
                Interceptor.attach(symbol.address, {
                    onEnter: function (args) {
                        this.arg0 = args[0];
                        this.arg1 = args[1];
                        this.arg2 = args[2];
                        LogPrint("go into " + Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol));
                        printNativeStack(this.context, Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol));
                        var size = ptr(this.arg2).toInt32();
                        if (size > 0) {
                            var sockfd = SSL_get_rfd(this.arg0);
                            var socketdetail = getsocketdetail(sockfd);
                            console.log(socketdetail + "---" + Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol) + hexdump(this.arg1, {
                                length: size
                            }));
                        }
                    }, onLeave(retval) {
                        LogPrint("leave " + Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol));

                    }
                });
            }
```

对于抹掉符号的可以直接hook read和write，然后在调用栈中回溯找到biowrite、SSL_write等函数 ，再定位到未加密前的函数，最后再进行hook

以下是frida提供的打印堆栈信息的接口，Fuzzy模式可以打印出更详细的信息

Fuzzy

![Untitled](https://i.loli.net/2021/11/05/8BEYWsQjybFcK2S.png)

ACCURATE

![Untitled](https://i.loli.net/2021/11/05/fSlLDn6cHjzFQo2.png)

完整代码 

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
    var trace = Thread.backtrace(context, Backtracer.FUZZY).map(DebugSymbol.fromAddress).join("\n");
    LogPrint("-----------start:" + name + "--------------");
    LogPrint(trace);
    LogPrint("-----------end:" + name + "--------------");

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
                var size = ptr(this.arg2).toInt32();
                if (size > 0) {
                    console.log(Process.getCurrentThreadId() + "---libc.so->write:" + hexdump(this.arg1, {
                        length: size
                    }));
                }
            }

        }, onLeave(retval) {
            LogPrint("leave libc.so->write");
        }
    });
}

function hookssl() {
    //SSL_get_rfd
    //int SSL_get_rfd(const SSL *ssl)
    var libsslmodule = Process.getModuleByName("libssl.so");
    var read_addr = libsslmodule.getExportByName("SSL_read");
    var write_addr = libsslmodule.getExportByName("SSL_write");
    var bioread_addr = libsslmodule.getExportByName("BIO_read");
    var biowrite_addr = libsslmodule.getExportByName("BIO_write");
    var SSL_get_rfd_ptr = libsslmodule.getExportByName('SSL_get_rfd');
    var SSL_get_rfd = new NativeFunction(SSL_get_rfd_ptr, 'int', ['pointer']);
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
                var sockfd = SSL_get_rfd(this.arg0);
                var socketdetail = getsocketdetail(sockfd);
                console.log(socketdetail + "---" + Process.getCurrentThreadId() + "---libssl.so->SSL_read:" + hexdump(this.arg1, {
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
            var size = ptr(this.arg2).toInt32();
            if (size > 0) {
                var sockfd = SSL_get_rfd(this.arg0);
                var socketdetail = getsocketdetail(sockfd);
                console.log(socketdetail + "---" + Process.getCurrentThreadId() + "---libssl.so->SSL_write:" + hexdump(this.arg1, {
                    length: size
                }));
            }
        }, onLeave(retval) {
            LogPrint("leave libssl.so->SSL_write");
        }
    });
    Interceptor.attach(bioread_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];
            LogPrint("go into libssl.so->bioread_addr");
            printNativeStack(this.context, Process.getCurrentThreadId() + "bioread_addr");
        }, onLeave(retval) {
            var size = retval.toInt32();
            if (size > 0) {
                var sockfd = SSL_get_rfd(this.arg0);
                var socketdetail = getsocketdetail(sockfd);
                console.log(socketdetail + "---" + Process.getCurrentThreadId() + "---libssl.so->bioread_addr:" + hexdump(this.arg1, {
                    length: size
                }));
            }
            LogPrint("leave libssl.so->bioread_addr");
        }
    });
    Interceptor.attach(biowrite_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];
            LogPrint("go into libssl.so->biowrite_addr");
            printNativeStack(this.context, Process.getCurrentThreadId() + "biowrite_addr");
            var size = ptr(this.arg2).toInt32();
            if (size > 0) {
                var sockfd = SSL_get_rfd(this.arg0);
                var socketdetail = getsocketdetail(sockfd);
                console.log(socketdetail + "---" + Process.getCurrentThreadId() + "---libssl.so->biowrite_addr:" + hexdump(this.arg1, {
                    length: size
                }));
            }
        }, onLeave(retval) {

            LogPrint("leave libssl.so->biowrite_addr");
        }
    });
}

function hookallssl() {
    var libsslmodule = Process.getModuleByName("libssl.so");
    var SSL_get_rfd_ptr = libsslmodule.getExportByName('SSL_get_rfd');
    var SSL_get_rfd = new NativeFunction(SSL_get_rfd_ptr, 'int', ['pointer']);
    //遍历所有so模块
    Process.enumerateModules().forEach(function (module) {
        //遍历所有符号,找到SSL_read或SSL_write后进行hook
        module.enumerateExports().forEach(function (symbol) {
            var name = symbol.name;
            if (name == 'SSL_read') {
                LogPrint(JSON.stringify(module) + JSON.stringify(symbol));
            }
            if (name == 'SSL_write') {
                LogPrint(JSON.stringify(module) + JSON.stringify(symbol));
                Interceptor.attach(symbol.address, {
                    onEnter: function (args) {
                        this.arg0 = args[0];
                        this.arg1 = args[1];
                        this.arg2 = args[2];
                        LogPrint("go into " + Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol));
                        printNativeStack(this.context, Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol));
                        var size = ptr(this.arg2).toInt32();
                        if (size > 0) {
                            var sockfd = SSL_get_rfd(this.arg0);
                            var socketdetail = getsocketdetail(sockfd);
                            console.log(socketdetail + "---" + Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol) + hexdump(this.arg1, {
                                length: size
                            }));
                        }
                    }, onLeave(retval) {
                        LogPrint("leave " + Process.getCurrentThreadId() + "---" + JSON.stringify(module) + "---" + JSON.stringify(symbol));

                    }
                });
            }

        })
    })

}

function main() {
    //对于抹掉符号的用这个函数
    //hooklibc();
    hookssl();
    //自编译openssl,函数符号还在的情况用这个函数
    //hookallssl();
}

setImmediate(main);
```