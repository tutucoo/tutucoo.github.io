---
title: 统一社会信用代码app过双向校验抓包
date: 2021-10-20 16:40:14
cover: https://i.loli.net/2021/10/20/CKRUBSziD5mawvr.jpg
---

hook找到证书密码并导出p12证书

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
    hook_ssl()
}
setImmediate(main);
```

![Untitled](https://i.loli.net/2021/10/20/lpPKmbI9aWZRFSt.png)

导入p12证书后仍然抓不到包，根据提示，客户端对服务器也进行了校验，可以通过hook ssl来绕过ssl pinning

![Untitled](https://i.loli.net/2021/10/20/9ye5Mn6VxGc3iSD.png)

对ssl进行hook，对所有重载函数都置空

```jsx
function hook_ssl() {
    Java.perform(function() {
        var ClassName = "com.android.org.conscrypt.Platform";
        var Platform = Java.use(ClassName);
        var targetMethod = "checkServerTrusted";
        var len = Platform[targetMethod].overloads.length;
        console.log(len);
        for(var i = 0; i < len; ++i) {
            Platform[targetMethod].overloads[i].implementation = function () {
                console.log("class:", ClassName, "target:", targetMethod, " i:", i, arguments);
                //printStack(ClassName + "." + targetMethod);
            }
        }
    });
}

function main(){
    hook_ssl()
}
setImmediate(main);
```

成功绕过双向校验

![Untitled](https://i.loli.net/2021/10/20/k35iGMmZQ9HjPn2.png)
