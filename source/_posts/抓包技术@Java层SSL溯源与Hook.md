---
title: 抓包技术@Java层SSL溯源与Hook
date: 2021-11-05 09:53:20
tags:
---

# Java层SSL溯源与Hook

对于一般的Socket通信，正常对收发函数进行hook就可以获取收发的内容，不过对于SSL流量就无法获取了，即使在调用传输层函数也是调用的connectTls，因此需要对ssl专门进行hook，从而获取未加密的内容

![Untitled](https://i.loli.net/2021/11/05/8hS65RizroNmlKM.png)

使用okhttp框架循环发送https请求 

```java
package com.example.ok_https;

import android.util.Log;

import com.squareup.okhttp.Call;
import com.squareup.okhttp.Callback;
import com.squareup.okhttp.FormEncodingBuilder;
import com.squareup.okhttp.OkHttpClient;
import com.squareup.okhttp.Request;
import com.squareup.okhttp.RequestBody;
import com.squareup.okhttp.Response;

import java.io.IOException;
import java.net.CookieManager;
import java.net.CookiePolicy;

public class Okhttp {
    public static void post(String url) {
        OkHttpClient okHttpClient = new OkHttpClient();
        okHttpClient.setCookieHandler(new CookieManager(null, CookiePolicy.ACCEPT_ALL));

        FormEncodingBuilder formEncodingBuilder = new FormEncodingBuilder();

        RequestBody requestBody = formEncodingBuilder.add("userName", "123").add("password", "123").build();
        Request.Builder builder = new Request.Builder();
        Request request = builder.url(url).post(requestBody).build();
        Call call = okHttpClient.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Request request, IOException e) {

            }

            @Override
            public void onResponse(Response response) throws IOException {
                if (!response.isSuccessful()) {
                    throw new IOException("Unexpected code " + response);
                } else if (response.isSuccessful()) {
                    Log.e("okhttp", "response.code()==" + response.code());
                    Log.e("okhttp", "response.message()==" + response.message());
                    Log.e("okhttp", "res==" + response.body().string());
                }
            }
        });

    }
}
```

AndroidManifest.xml文件中添加权限 

```java
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.INTERNET"/>
```

在build.gradle中添加okhttp依赖

![Untitled](https://i.loli.net/2021/11/05/ZyBhxEK8Fksc4lr.png)

在MainActivity中循环发送https请求 

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    Okhttp.post("https://www.baidu.com");
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

            }
        }).start();
    }
}
```

进行动态调试溯源

发送数据: 

```java
com.android.org.conscrypt.OpenSSLSocketImpl$SSLOutputStream 

public void write(int oneByte); 

public void write(byte[] buf,int offset,int byteCount)

public static native void SSL_write(long sslNativePointer,FileDescriptor fd,SSLHandshakeCallbacks hsc,byte[] b,int off,int len, int write)

NativeCrypto.SSL_write;
```

接收数据: 

```java
com.android.org.conscrypt.OpenSSLSocketImpl$SSLInputStream

public int read()

public int read(byte[] buf,int offset,int byteCount)

org.conscrypt.NativeCrypto->SSL_read

public static native int SSL_read()
```

如果Java层hook不到任何数据，有可能是多进程，如果多进程也hook不到数据，那通信就是在JNI层进行处理的

frida脚本，注意：不同Android版本api可能不一样

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

function hooksslsocket() {
    Java.perform(function () {
        var SSLInputStreamClass = Java.use('com.android.org.conscrypt.ConscryptFileDescriptorSocket$SSLInputStream');
        //public int read(byte[] buf, int offset, int byteCount)
        SSLInputStreamClass.read.overload().implementation = function () {
            var value = this.read();
            var sslsocketimplwrapper = this.this$0.value;
            console.log("SSLInputStreamClass.read 1obj->" + sslsocketimplwrapper);
            var socket = sslsocketimplwrapper.socket.value;
            console.log("SSLInputStreamClass.read 1socket->" + socket);
            //console.log("[" + Process.getCurrentThreadId() + "]SSLInputStreamClass.read:len:" + 4 + "--content:" + JSON.stringify(value));
            console.log("[" + Process.getCurrentThreadId() + "]sslsocket read a int:" + value);
            printJavaStack('sslInputStream.read()')
            return value;
        }

        SSLInputStreamClass.read.overload('[B', 'int', 'int').implementation = function (arg0, arg1, arg2) {
            var size = this.read(arg0, arg1, arg2);
            //内部类对象直接得到外部类对象的方法
            var sslsocketimplwrapper = this.this$0.value;
            console.log("SSLInputStreamClass.read 2obj->" + sslsocketimplwrapper);
            var socket = sslsocketimplwrapper.socket.value;
            console.log("SSLInputStreamClass.read 2socket->" + socket);
            //console.log("[" + Process.getCurrentThreadId() + "]SSLInputStreamClass.read:len:" + size + "--content:" + JSON.stringify(arg0));
            var bytearray = Java.array('byte', arg0);
            var content = '';
            for (var i = 0; i < size; i++) {
                if (isprintable(bytearray[i])) {
                    content = content + String.fromCharCode(bytearray[i]);
                }
            }
            /*            var socketimpl = this.impl.value;
                        var address = socketimpl.address.value;
                        var port = socketimpl.port.value;*/
            console.log("[" + Process.getCurrentThreadId() + "]sslsocket read:" + content);
            //console.log("\n" + JSON.stringify(this.socket.value) + "\n[" + Process.getCurrentThreadId() + "]send:" + content);
            printJavaStack('sslInputStream.read')
            return size;
        }

        var SSLOutputStreamClass = Java.use('com.android.org.conscrypt.ConscryptFileDescriptorSocket$SSLOutputStream');
        SSLOutputStreamClass.write.overload('in
t').implementation = function (arg0) {
            var result = this.write(arg0);
            //通过内部类对象得到外部类对象
            var sslsocketimplwrapper = this.this$0.value;
            console.log("SSLOutputStream 1obj->" + sslsocketimplwrapper);
            var socket = sslsocketimplwrapper.socket.value;
            console.log("SSLOutputStream 1socket->" + socket);
            /*SSLOutputStream 2obj->SSL socket over Socket[address=portal.meirishanglai.com/240.0.0.21,port=443,localPort=37563]
              SSLOutputStream 2socket->Socket[address=portal.meirishanglai.com/240.0.0.21,port=443,localPort=37563]
            */
            console.log("[" + Process.getCurrentThreadId() + "]write(int):len:" + 4 + "--content:" + arg0);
            printJavaStack('sslOutputStream.write(int)')
            return result;
        }

        //public void write(byte[] buf, int offset, int byteCount)
        SSLOutputStreamClass.write.overload('[B', 'int', 'int').implementation = function (arg0, arg1, arg2) {
            var result = this.write(arg0, arg1, arg2);
            var sslsocketimplwrapper = this.this$0.value;
            console.log("SSLOutputStream 2obj->" + sslsocketimplwrapper);
            var socket = sslsocketimplwrapper.socket.value;
            console.log("SSLOutputStream 2socket->" + socket);
            //console.log("[" + Process.getCurrentThreadId() + "]socketWrite0:len:" + arg2 + "--content:" + JSON.stringify(arg0));
            var bytearray = Java.array('byte', arg0);
            var content = '';
            for (var i = 0; i < arg2; i++) {
                if (isprintable(bytearray[i])) {
                    content = content + String.fromCharCode(bytearray[i]);
                }

            }
            /*            var socketimpl = this.impl.value;
                        var address = socketimpl.address.value;
                        var port = socketimpl.port.value;*/
            console.log("[" + Process.getCurrentThreadId() + "]sslsocket send:" + content);
            //console.log("\n" + JSON.stringify(this.socket.value) + "\n[" + Process.getCurrentThreadId() + "]send:" + content);
            printJavaStack('sslOutputStream.write')
            return result;
        }
    })
}

function main() {
    hooksslsocket();
}

setImmediate(main)
```

![Untitled](https://i.loli.net/2021/11/05/Vzj3GTe7cZP1yFW.png)