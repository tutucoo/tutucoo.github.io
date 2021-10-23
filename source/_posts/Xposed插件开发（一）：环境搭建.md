---
title: Xposed插件开发（一）：环境搭建
date: 2021-10-23 16:22:12
tags:
---

# **Xposed简介**

通过替换/system/bin/app_process程序控制zygote进程，使得app_process在启动过程中会加载XposedBridge.jar这个jar包，从而完成对Zygote进程及其创建的Dalvik虚拟机的劫持。

# **环境搭建**

dalvik环境下安装比较简单，直接下载xposedinstaller安装就可以了，因为年代久远，dalvik环境已经很少见了。

ART环境下安装需要下载Xposed*sdk*.zip文件和XposedInstaller.apk文件，zip文件需要在recovery mode中刷入，XposedInstaller是模块管理工具，直接安装就可以使用。

Xposed正式版最后支持到7.1，7.1以后的就不再支持了，现在一般使用EdXposed或者LSPosed作为替代。

EdXposed和LSPosed安装都非常简单，在Magisk app搜索安装就可以，这两个框架都依赖于riru框架，riru同样也是在Magisk app里进行下载安装 。

# **基于Xposed的插件框架**

Xposed的受欢迎程度下降的主要原因是Magisk，它提供了“无系统”的方法。Xposed Framework修改了Android系统，从而触发Google SafetyNet禁用诸如Google Pay，Netflix和Pokemon GO之类的功能。另一方面，Magisk不会修改系统。它使用引导分区而不是系统。当请求系统文件时，Magisk会在其位置覆盖“虚拟文件”。

现在Xposed可以与Magisk一起使用。Xposed Framework可以作为Magisk模块安装。这意味着Xposed也可以是无系统的，您可以在不触发Google SafetyNet的情况下使用mod。

## 简单案例

1、导入XposedBridge
放入项目libs文件夹中并add as library

![Untitled](https://i.loli.net/2021/10/23/Hc7lxBud3oXja2N.png)

2、设置libs为`compileOnly`

因为xposedBridgeApi-54.jar是很老的库，如果设置为implementation就会被编译进app，app就会报错

![Untitled](https://i.loli.net/2021/10/23/glhk1NnHRfbWou3.png)

修改如下

![Untitled](https://i.loli.net/2021/10/23/9hgNyz4AePlSkEq.png)

3、在AndroidManifest.xml中声明元数据

```xml
 <?xml version="1.0" encoding="utf-8"?>
 <manifest xmlns:android="http://schemas.android.com/apk/res/android"
     package="de.robv.android.xposed.mods.tutorial"
     android:versionCode="1"
     android:versionName="1.0" >
 
     <uses-sdk android:minSdkVersion="15" />

     <application
         android:icon="@drawable/ic_launcher"
         android:label="@string/app_name" >
         <meta-data
             android:name="xposedmodule"
             android:value="true" />
         <meta-data
             android:name="xposeddescription"
             android:value="Easy example which makes the status bar clock red and adds a smiley" />
         <meta-data
             android:name="xposedminversion"
             android:value="53" />
     </application>
 </manifest>
```

4、编写hook代码

新建一个Test文件

![Untitled](https://i.loli.net/2021/10/23/7XyTYVldNutHURA.png)

```java
public class Test implements IXposedHookLoadPackage {
    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam loadPackageParam) throws Throwable {
				//这里会打印所有加载的包名，包括系统的apk
        Log.i("tutucoo", "handleLoadPackage: " + loadPackageParam.packageName);
				//会在xposed管理apk的日志里看到打印的所有系统包名
        XposedBridge.log("Test->" + loadPackageParam.packageName);
				//这里对目标apk进行hook，在进入apk时如果正确hook，会在xposed管理器apk的日志处看到下面的日志（此时logcat选择目标apk也会打印Xposed日志
        if (loadPackageParam.packageName.equals("com.example.simpleencryption")){
            XposedBridge.log("--------------packageName is com.example.simpleencryption------------");
        }

    }
}
```

5、编写xposed_init文件
在main文件夹下新建文件夹assets，再新建文件xposed_init指定入口

```
 包名+类名
```

![Untitled](https://i.loli.net/2021/10/23/cqfYWZ1N5Ed2XnH.png)

如果有多个hook文件，分多行声明，哪个先声明，哪个就先生效

6、编译并安装模块

编译apk，安装到手机，在Xposed管理器中启用模块，重启

此时应该可以在logcat中看到打印的日志了

![Untitled](https://i.loli.net/2021/10/23/DJwZWYmASk26gIX.png)

logcat同样也会打印xposedbridge.log打印的日志

![Untitled](https://i.loli.net/2021/10/23/fKr23ZMAapEgcTG.png)
