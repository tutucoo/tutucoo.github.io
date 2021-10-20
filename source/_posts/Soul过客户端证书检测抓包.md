---
title: Soul过客户端证书检测抓包
date: 2021-10-20 15:45:18
cover: https://i.loli.net/2021/10/20/Zn8BCOv5G4zf3wy.jpg
---

使用charles对soul进行抓包提示服务器拒绝连接，猜测是服务端通过p12证书对客户端进行校验 

![Untitled](https://i.loli.net/2021/10/20/tDvJ7XKrYCh1Hwu.png)

一般这种情况使用frida脚本对KeyStore进行hook，把p12写到sdcard中并获取密钥

```jsx
function hook_KeyStore_load() {
    Java.perform(function () {
        var ByteString = Java.use("com.android.okhttp.okio.ByteString");
        var myArray=new Array(1024);
        var i = 0
        for (i = 0; i < myArray.length; i++) {
            myArray[i]= 0x0;
         }
        var buffer = Java.array('byte',myArray);
        
        var StringClass = Java.use("java.lang.String");
        var KeyStore = Java.use("java.security.KeyStore");
        KeyStore.load.overload('java.security.KeyStore$LoadStoreParameter').implementation = function (arg0) {
            console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));

            console.log("KeyStore.load1:", arg0);
            this.load(arg0);
        };
        KeyStore.load.overload('java.io.InputStream', '[C').implementation = function (arg0, arg1) {
            console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));

            console.log("KeyStore.load2:", arg0, arg1 ? StringClass.$new(arg1) : null);

            if (arg0){
                var file =  Java.use("java.io.File").$new("/sdcard/Download/"+ String(arg0)+".p12");
                var out = Java.use("java.io.FileOutputStream").$new(file);
                var r;
                while( (r = arg0.read(buffer)) > 0){
                    out.write(buffer,0,r)
                }
                console.log("save success!")
                out.close()
            }
            this.load(arg0, arg1);
        };

        console.log("hook_KeyStore_load...");

    });
}

function main(){
    hook_KeyStore_load()    
}
setImmediate(main);
```

成功hook后，会将p12文件写入到sdcard中，adb pull出来就可以了，如果pull失败可以对p12文件进行压缩

这里获取到密钥是：}%2R+\OSsjpP!w%X

![Untitled](https://i.loli.net/2021/10/20/SWZEcYnIhD7XFpd.png)

进入Charles，打开SSL Proxying Settings，添加一个客户端证书，导入p12并按要求输入密码

之后再设置主机白名单

![Untitled](https://i.loli.net/2021/10/20/ymnVWHKPJhSt2i1.png)

成功抓取soul数据包

![Untitled](https://i.loli.net/2021/10/20/KiRnWfM7cLtVm6C.png)

![Untitled](https://i.loli.net/2021/10/20/zHeOjyfxQgqVmXL.png)
