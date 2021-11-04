---
title: jni层socket溯源与Hook
date: 2021-11-04 22:18:25
tags:
---

# jni socket溯源

java层socket请求最后调用的api是socketRead0和socketWrite0

在Android8.0源码中搜索socketRead0，找到所在的c文件，Android会通过编译c文件生成一个so文件

![Untitled](https://i.loli.net/2021/11/04/TvgZuIUOFmQKbCA.png)

通过这个函数的函数名看出它是一个动态注册的函数，因此找到so文件以后要通过JNI_OnLoad函数找到注册函数的位置

![Untitled](https://i.loli.net/2021/11/04/qlk5aNvuTeZsgzA.png)

SocketInputStream.c所在的文件夹是/libcore/ojluni，在该文件夹下通过搜索找到它，可以看到同时找到了一个openjdksub.mk，该文件会对SocketOutputstream.c进行编译并生成so文件，不过打开openjdksub.mk并没有看到生成so文件的配置 

![jni%E5%B1%82socket%E6%BA%AF%E6%BA%90%E4%B8%8EHook%20d4893f5d804244cfaff2d74c07da4008/Untitled%202.png](https://i.loli.net/2021/11/04/iXJ5apvKQO1Sdh4.png)

退到上一层目录，继续搜索openjdksub.mk，可以找到NativeCode.mk文件

![Untitled](https://i.loli.net/2021/11/04/Sx1Znv64qQIsP8J.png)

可以看到编译的so文件名，那么最终SocketOutputStream.c会被编译到libopenjdk.so中，所以接下来用IDA逆向libopenjdk.so，就可以找到socketRead0动态注册的代码

![Untitled](https://i.loli.net/2021/11/04/YoGqtCfHeQ2OdVy.png)

生成的so位于/system/lib/libopenjdk.so，通过adb pull可以获取到文件，拖入到IDA中进行分析

在JNI_OnLoad中找到SocketInputStream和SocketOutputStream注册的代码

![Untitled](https://i.loli.net/2021/11/04/c4JeubdDPm9ZzkF.png)

进入SocketInputStream找到jniRegisterNativeMethods，这个函数是用来动态注册jni函数的，不过这里解析有点错误，没有看到它的参数

![Untitled](https://i.loli.net/2021/11/04/TgFXdIwWm6zfSRJ.png)

这个函数一般的用法，可以看到除了JNIEnv*作为第一个参数，还有3个参数，分别是类名、函数数组、函数数量

```java
extern "C" int jniRegisterNativeMethods(C_JNIEnv* env, const char* className, const JNINativeMethod* gMethods, int numMethods) {
    JNIEnv* e = reinterpret_cast<JNIEnv*>(env);
    scoped_local_ref<jclass> c(env, findClass(env, className));
    if (c.get() == NULL) {
        e->FatalError("");
    }
    if ((*env)->RegisterNatives(e, c.get(), gMethods, numMethods) < 0) {
        e->FatalError("");
    }
    return 0;
}
```

按tab键查看反汇编 

![Untitled](https://i.loli.net/2021/11/04/Qm5OfNWXjVGe9qi.png)

跳转到off_287c4可以看到是函数数组，这也证实了这个函数确实传递了参数

![Untitled](https://i.loli.net/2021/11/04/KngmHfTh2euM7Zk.png)

进行还原参数

![Untitled](https://i.loli.net/2021/11/04/knfO8lFNr6AbxBy.png)

搜索到socketWrite0函数，然后可以查看xrefs from，看到它接下来会调用j_NET_Send函数  

![Untitled](https://i.loli.net/2021/11/04/aGVZPTwc5pKJdMv.png)

在socketWrite0中找到j_NET_Send函数进入，调用了NET_Send函数 

![Untitled](https://i.loli.net/2021/11/04/ZMs3i6YwVxuAbjX.png)

然后是sendto函数  

![Untitled](https://i.loli.net/2021/11/04/8Se4Zx5ycdObfo3.png)

可以看到sendto函数是导入的，它位于libc.so文件中，sendto内部调用了syscall，完成最终调用 

![Untitled](https://i.loli.net/2021/11/04/oQPXy5RMAnreuzv.png)

# hook jni层tcp socket接口

先搭建tcp通信环境，然后进行frida hook

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

function hooklibc() {
    var libcmodule = Process.getModuleByName("libc.so");
    var recvfrom_addr = libcmodule.getExportByName("recvfrom");
    var sendto_addr = libcmodule.getExportByName("sendto");
    console.log(recvfrom_addr + "---" + sendto_addr);
    //ssize_t recvfrom(int fd, void *buf, size_t n, int flags, struct sockaddr *addr, socklen_t *addr_len)
    Interceptor.attach(recvfrom_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];

            LogPrint("go into libc.so->recvfom");

            printNativeStack(this.context, "recvfom");
        }, onLeave(retval) {
            var size = retval.toInt32();
            if (size > 0) {
                var result = getsocketdetail(this.arg0.toInt32());
                console.log(result + "---libc.so->recvfrom:" + hexdump(this.arg1, {
                    length: size
                }));
            }

            LogPrint("leave libc.so->recvfom");
        }
    });
    //ssize_t sendto(int fd, const void *buf, size_t n, int flags, const struct sockaddr *addr, socklen_t addr_len)
    Interceptor.attach(sendto_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];
            LogPrint("go into libc.so->sendto");
            printNativeStack(this.context, "sendto");
        }, onLeave(retval) {
            var size = ptr(this.arg2).toInt32();
            if (size > 0) {
                var result = getsocketdetail(this.arg0.toInt32());
                console.log(result + "---libc.so->sendto:" + hexdump(this.arg1, {
                    length: size
                }));
            }

            LogPrint("leave libc.so->sendto");
        }
    });
}

function main() {
    hooklibc();
}

setImmediate(main);
```

![Untitled](https://i.loli.net/2021/11/04/on9DlVCESTuxU1s.png)

# hook jni层udp socket接口

跟hook tcp会有些差别，因为在实际调试过程中，发现第一个参数套接字，已经无法从中得到ip和端口了，所以只能从第4个参数进行hook，取出来，发现是一个结构体，一个字节一个字节的读取出来，然后再本地计算，拼接，就ok了。

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
    //frida自带接口
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

function getudpaddr(addrptr) {
    var port_ptr = addrptr.add(2);
    var port = ptr(port_ptr).readU8() * 256 + ptr(port_ptr.add(1)).readU8();
    var ip_ptr = addrptr.add(4);
    var ip_addr = getip(ip_ptr);
    return "peer:"+ip_addr+"--port:"+port;
}

function hooklibc() {
    var libcmodule = Process.getModuleByName("libc.so");
    var recvfrom_addr = libcmodule.getExportByName("recvfrom");
    var sendto_addr = libcmodule.getExportByName("sendto");
    console.log(recvfrom_addr + "---" + sendto_addr);
    //ssize_t recvfrom(int fd, void *buf, size_t n, int flags, struct sockaddr *addr, socklen_t *addr_len)
    Interceptor.attach(recvfrom_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];
            this.arg3 = args[3];
            this.arg4 = args[4];
            this.arg5 = args[5];
            LogPrint("go into libc.so->recvfom");

            printNativeStack(this.context, "recvfom");
        }, onLeave(retval) {
            var size = retval.toInt32();
            if (size > 0) {
                var result = getsocketdetail(this.arg0.toInt32());
                if (result.indexOf('udp') >= 0) {
                    /*75struct sockaddr_in {
                    76	short	sin_family;
                    77	u_short	sin_port;
                    78	struct in_addr	sin_addr;
                    79	char	sin_zero[8];
                    80};*/
                    var sockaddr_in_ptr = this.arg4;
                    var sizeofsockaddr_in = this.arg5;

                    //02 00 22 b8 c0 a8 05 96 00 00 00 00 00 00 00 00
                    console.log("this is a recvfrom udp!->" + getudpaddr(sockaddr_in_ptr) + "---" + sizeofsockaddr_in);
                }
                console.log(Process.getCurrentThreadId()+result + "---libc.so->recvfrom:" + hexdump(this.arg1, {
                    length: size
                }));
            }

            LogPrint("leave libc.so->recvfom");
        }
    });
    //ssize_t sendto(int fd, const void *buf, size_t n, int flags, const struct sockaddr *addr, socklen_t addr_len)
    Interceptor.attach(sendto_addr, {
        onEnter: function (args) {
            this.arg0 = args[0];
            this.arg1 = args[1];
            this.arg2 = args[2];
            this.arg3 = args[3];
            this.arg4 = args[4];
            this.arg5 = args[5];
            LogPrint("go into libc.so->sendto");
            printNativeStack(this.context, "sendto");
        }, onLeave(retval) {
            var size = ptr(this.arg2).toInt32();
            if (size > 0) {
                var result = getsocketdetail(this.arg0.toInt32());
                if (result.indexOf('udp') >= 0) {
                    /*75struct sockaddr_in {
                    76	short	sin_family;
                    77	u_short	sin_port;
                    78	struct in_addr	sin_addr;
                    79	char	sin_zero[8];
                    80};*/
                    var sockaddr_in_ptr = this.arg4;
                    var sizeofsockaddr_in = this.arg5;

                    //02 00 22 b8 c0 a8 05 96 00 00 00 00 00 00 00 00
                    console.log("this is a sendto udp!->" + getudpaddr(sockaddr_in_ptr) + "---" + sizeofsockaddr_in);
                }
                console.log(Process.getCurrentThreadId()+"---"+result + "---libc.so->sendto:" + hexdump(this.arg1, {
                    length: size
                }));
            }

            LogPrint("leave libc.so->sendto");
        }
    });
}

function main() {
    hooklibc();
}

setImmediate(main);
```

![Untitled](https://i.loli.net/2021/11/04/qIg7xHFDLiumoRw.png)

# 测试native socket代码

在native层使用socket发送一个简单的http请求，再使用frida hook jni层的socket接口

httpget.h

```java
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <ctype.h>
#include <unistd.h>
#include <netdb.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int GetHttpResponseHead(int sock,char *buf,int size);
int HttpGet(const char *server,const char *url);
int testhttp(void);
```

httpget.c

```java
#include "httpget.h"
#include <android/log.h>

int GetHttpResponseHead(int sock, char *buf, int size) {
    int i;
    char *code, *status;
    memset(buf, 0, size);
    for (i = 0; i < size - 1; i++) {
        if (recv(sock, buf + i, 1, 0) != 1) {
            perror("recv error");
            return -1;
        }
        if (strstr(buf, "\r\n\r\n"))
            break;
    }
    if (i >= size - 1) {
        puts("Http response head too long.");
        return -2;
    }
    code = strstr(buf, " 200 ");
    status = strstr(buf, "\r\n");
    if (!code || code > status) {
        *status = 0;
        printf("Bad http response:\"%s\"\n", buf);
        return -3;
    }
    return i;
}

int HttpGet(const char *server, const char *url) {
    __android_log_print(ANDROID_LOG_INFO, "-----from--jni-------", "Enter HttpGet function!");
    int sock = socket(PF_INET, SOCK_STREAM, 0);
    __android_log_print(ANDROID_LOG_INFO, "-----from--jni-------", "%d", sock);
    struct sockaddr_in peerAddr;
    char buf[2048];
    int ret;
    //const char *filename;
    //FILE *fp=NULL;
    peerAddr.sin_family = AF_INET;
    peerAddr.sin_port = htons(5000);
    peerAddr.sin_addr.s_addr = inet_addr("192.168.5.150");
    ret = connect(sock, (struct sockaddr *) &peerAddr, sizeof(peerAddr));
    if (ret != 0) {
        perror("connect failed");
        close(sock);
        return -1;
    }
    sprintf(buf,
            "GET %s HTTP/1.1\r\n"
            "Accept: */*\r\n"
            "User-Agent: FireFox\r\n"
            "Host: %s\r\n"
            "Connection: Close\r\n\r\n",
            url, server);
    send(sock, buf, strlen(buf), 0);
    __android_log_print(ANDROID_LOG_INFO, "-----from--jni-------", "%s", "send over");

    if (GetHttpResponseHead(sock, buf, sizeof(buf)) < 1) {
        close(sock);
        return -1;
    }
    __android_log_print(ANDROID_LOG_INFO, "-----from--jni-------",
                        "Enter HttpGet function's while!");
    while ((ret = recv(sock, buf, sizeof(buf), 0)) > 0) {
        __android_log_print(ANDROID_LOG_INFO, "-----from--jni-------",
                            "hello, this is my number %s log message!", buf);
    }
    //}
    //fclose(fp);
    shutdown(sock, SHUT_RDWR);
    close(sock);
    return 0;
}

//"http://192.168.7.222/index.html"
int testhttp(void) {
    __android_log_print(ANDROID_LOG_INFO, "-----from--jni-------", "Enter mmain function!");
    char *head, *tail;
    char server[128] = {0};
    head = strstr("http://192.168.1.141:5000/index.html", "//");
    if (!head) {
        puts("Bad url format");
        return -1;
    }
    head += 2;
    tail = strchr(head, '/');
    if (!tail) {
        return HttpGet(head, "/");
    } else if (tail - head > sizeof(server) - 1) {
        puts("Bad url format");
        return -1;
    } else {
        memcpy(server, head, tail - head);
        return HttpGet(server, tail);
    }
}
```

创建一个线程发送http请求

```java
#include <jni.h>
#include <stdio.h>
#include <string.h>
#include <android/log.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <android/log.h>
#include <unistd.h>
#include <netdb.h>
#include <pthread.h>

extern "C" {
#include "httpget.h"
}
#define  LOG_TAG    "socket"

#define  LOGI(...)  __android_log_print(ANDROID_LOG_INFO,LOG_TAG,__VA_ARGS__)
#define MAXSIZE 4096
pthread_t pthread;//线程对象

void *threadDoThings(void *data) {
    LOGI("jni thread do things");
    while (true) {
        pthread_t udppthread,tcpthread;//线程对象
        pthread_create(&tcpthread, NULL, reinterpret_cast<void *(*)(void *)>(testhttp), NULL);
        sleep(5);
    }

    //pthread_//exit(&pthread);
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_okhttp_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    // TODO
    LOGI("normal thread");
    //创建线程
    pthread_create(&pthread, NULL, threadDoThings, NULL);
    const char *hello = "Hello from C++";
    return env->NewStringUTF(hello);
}
```

AndroidManifest.xml授权 

```java
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.INTERNET"/>
```

启动web服务端，再运行程序，最后使用之前的tcp hook代码进行hook 

![Untitled](https://i.loli.net/2021/11/04/hAHnc1mN5kxUWaF.png)