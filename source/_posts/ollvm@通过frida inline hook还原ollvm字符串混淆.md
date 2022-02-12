---
title: ollvm@通过frida inline hook还原ollvm字符串混淆
date: 2022-02-11 17:10:37
tags:
---
# 通过frida inline hook还原ollvm字符串混淆

还是在init_array段里，发现有几个函数，进去看一下

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled.png](https://s2.loli.net/2022/02/11/EjZhxfoHm98pFSa.png)

发现这些都不是解密字符串的函数 

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%201.png](https://s2.loli.net/2022/02/11/9gmLi4ShtQ1RjJT.png)

在JNI_Onload中给参数转类型JavaVM*会出现bug，需要指定jni头文件，7.0以后会自动解析

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%202.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211j1klBV.png)

进入JNI_OnLoad函数，看到数组偏移v9是被xmmword_3E1E8赋值的

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%203.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211erhr3N.png)

向上回溯，找到数组偏移的最终来源dword_3E1B4

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%204.png](https://gitee.com/tutucoo/images/raw/master/uPic/202202111p4kan.png)

再对它进行转码，转为ngis，这就是函数的名字sign的倒序字母

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%205.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211coVaLi.png)

xmmword_3e1e8跟dword_3e184是一样的，所以下面的是对参数的赋值 ，v6就是参数，而v6又是byte_3e1ba赋值的，byte_3e1ba的值来自byte_22e80，这个值就是函数的参数

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%206.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211NMTrVg.png)

hook byte_3e1ba的值，发现它也是一个字符串 

看下byte_22e80的值 

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%207.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211kpbmfx.png)

byte_22e80的值赋值给byte_3e195

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%208.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211WUUPGZ.png)

为了知道RegisterNatives函数传递的参数的具体内容，对RegisterNatives函数进行hook

每个参数大小是8个字节，因为是64位系统

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%209.png](https://gitee.com/tutucoo/images/raw/master/uPic/202202112ZHfj7.png)

动态注册的函数名是sign1，返回值是一个字符串，也就是说只要能hook到RegisterNatives函数就能找到动态注册的函数名

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/202202116dseOl.png)

如果碰到一种情况，无法找到RegisterNatives函数的地址，就比较麻烦了

我们可以查看byte_22e80的反汇编 

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%2011.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211rEHoXG.png)

731c进行了异或，可以hook 7320，就可以解密的字符串了

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%2012.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211tnM0Yp.png)

```jsx
function inline_hook(){
    var base_hello_jni = Module.findBaseAddress("libhello-jni.so");
    console.log(base_hello_jni);
    if(base_hello_jni){
        console.log(base_hello_jni);
        var addr_07320 = base_hello_jni.add(0x07320);
        // inline hook 和 hook函数是一样的
        Interceptor.attach(addr_07320,{
            onEnter:function(args){
                // 
                // 64位: X0-X30, XZR(零寄存器)   要学汇编! 不然没法玩inline hook
                console.log("addr_07320 x13",this.context.x13, " x14",this.context.x14);
            },
            onLeave:function(retval){}
        })

        
    
    }
}
```

然后我们会发现没法hook成功，是因为frida注入脚本的时候，我们要hook的so文件还没加载。这时候我们需要来hook dlopen,dlopen加载成功以后在去执行inline_hook

```jsx
void * dlopen（const char * filename ，int flag ）;
```

dlopen有2个参数, 我们打印第一个 filename

```jsx
function hook_dlopen(){

    var dlopen = Module.findExportByName(null,"dlopen");
    Interceptor.attach(dlopen,{
        onEnter:function(args){
            console.log("dlopen",ptr(args[0]).readCString());
        },
        onLeave:function(retval){}
    })
}
```

发现没有我们需要的.

```jsx
[Pixel 2::com.example.hellojni_sign2]-> dlopen /vendor/lib64/egl/libGLESv2_adreno.so
dlopen /vendor/lib64/egl/libEGL_adreno.so   
dlopen /vendor/lib64/egl/libGLESv2_adreno.so
dlopen /system/lib64/libEGL.so
dlopen /system/lib64/libGLESv2.so
dlopen /system/lib64/libGLESv1_CM.so
dlopen /vendor/lib64/egl/eglSubDriverAndroid.so
dlopen libadreno_utils.so
dlopen /vendor/lib64/egl/libEGL_adreno.so
```

在android6.0的时候,hookdlopen的时候是可以加载所有的so的,但是在高版本的时候就没有了, 我们需要hook另外一个函数,叫做android_dlopen_ext, 所以我们去hook他

```jsx
function hook_dlopen(){

    var dlopen = Module.findExportByName(null,"dlopen");
    Interceptor.attach(dlopen,{
        onEnter:function(args){
            console.log("dlopen",ptr(args[0]).readCString());
        },
        onLeave:function(retval){}
    });

    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            console.log("android_dlopen_ext",ptr(args[0]).readCString());
        },
        onLeave:function(retval){}
    });
}
```

返回结果, 我们可以看到libhello-jni.so 那么此时就可以去hook他了

```jsx
Spawned `com.example.hellojni_sign2`. Resuming main thread!
[Pixel 2::com.example.hellojni_sign2]-> android_dlopen_ext /data/app/com.example.hellojni_sign2-VfsaSmMy0zmiYY_Le5Ylfw==/oat/arm64/base.odex
android_dlopen_ext /vendor/lib64/egl/libEGL_adreno.so
dlopen /vendor/lib64/egl/libGLESv2_adreno.so
android_dlopen_ext /data/app/com.example.hellojni_sign2-VfsaSmMy0zmiYY_Le5Ylfw==/lib/arm64/libhello-jni.so
dlopen /vendor/lib64/egl/libEGL_adreno.so
android_dlopen_ext /vendor/lib64/egl/libGLESv1_CM_adreno.so
dlopen /vendor/lib64/egl/libGLESv2_adreno.so
android_dlopen_ext /vendor/lib64/egl/libGLESv2_adreno.so
dlopen /system/lib64/libEGL.so
dlopen /system/lib64/libGLESv2.so
dlopen /system/lib64/libGLESv1_CM.so
dlopen /vendor/lib64/egl/eglSubDriverAndroid.so
android_dlopen_ext /vendor/lib64/hw/gralloc.msm8998.so
dlopen libadreno_utils.so
dlopen /vendor/lib64/egl/libEGL_adreno.so
android_dlopen_ext /vendor/lib64/hw/android.hardware.graphics.mapper@2.0-impl.so
android_dlopen_ext /vendor/lib64/hw/gralloc.msm8998.so
```

然后再修改下代码,整体代码大概如下

```jsx
function inline_hook(){
    var base_hello_jni = Module.findBaseAddress("libhello-jni.so");
    console.log(base_hello_jni);
    if(base_hello_jni){
        console.log("base_hello_jni",base_hello_jni);
        var addr_07320 = base_hello_jni.add(0x07320);
        // inline hook 和 hook函数是一样的
        Interceptor.attach(addr_07320,{
            onEnter:function(args){
                // 
                // 根据反汇编值保存在W13，在ARM64里面x13的第8位是W13
                console.log("addr_07320 x13",this.context.x13);
            },
            onLeave:function(retval){}
        })

    
    }
}
var is_hook_hello_jni = true;
function hook_dlopen(){

    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            var so_name = ptr(args[0]).readCString();
            if(so_name.indexOf("libhello-jni.so")){
                is_hook_hello_jni = true
            }
            console.log("android_dlopen_ext",so_name);
        },
        onLeave:function(retval){
            if(is_hook_hello_jni){
                inline_hook()
            }
        }
    });
}
```

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/20220211Ns7UMD.png)

再把x13的值放到010editor中查看.就可以看到签名了.

![%E9%80%9A%E8%BF%87frida%20inline%20hook%E8%BF%98%E5%8E%9Follvm%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B7%B7%E6%B7%86%202294b779590f423ea72d4d4df0268c01/Untitled%2014.png](https://gitee.com/tutucoo/images/raw/master/uPic/20220211VGOj2I.png)