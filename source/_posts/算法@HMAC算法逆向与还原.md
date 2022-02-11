---
title: 算法@HMAC算法逆向与还原
date: 2022-02-11 14:57:37
tags:
---
# HMAC算法逆向与还原

## HMAC算法还原

HMAC有一个key，并且可以选择多种不同的哈希算法

目标程序对字符串进行加密，算法是HMAC

![Untitled](https://s2.loli.net/2022/02/11/dwgu7GhSn4RkxEV.png)

核心加密函数，传递了一个字符串kanxue_imyang

![Untitled](https://s2.loli.net/2022/02/11/Z728GloLjqYsNn4.png)

翻算法找到常量

![Untitled](https://s2.loli.net/2022/02/11/vpjFLcUTuQ4Y6rC.png)

然后google搜到算法源码，根据源码可以知道这个函数是md5_transform

![Untitled](https://s2.loli.net/2022/02/11/tA7gdIKsjTu864M.png)

它的上级函数是md5_update

![Untitled](https://s2.loli.net/2022/02/11/MJke5gPATpEUcKu.png)

地址是0xaf84

![Untitled](https://s2.loli.net/2022/02/11/CURcih9sAHrJV6y.png)

对md5_update函数进行hook

```c
function test(){
    var base_native_lib = Module.findBaseAddress("libnative-lib.so");
    console.log("base_native_lib address:",base_native_lib);
    var md5_update = base_native_lib.add(0xAF84 + 1);
    console.log("md5_update address:",md5_update);

    Interceptor.attach(md5_update,{
        onEnter : function(args){
            console.log("md5_update:",hexdump(args[1],{length : parseInt(args[2])}),args[0]);
        },onLeave : function(retval){

        }
    })
}

setTimeout(test,3000);
```

每次加密都会调用8次md5_update函数 

![Untitled](https://s2.loli.net/2022/02/11/YQLyX6a9t3RF45H.png)

打印一下md5_update调用堆栈，大部分上层函数都指向了0xb738

接着hook sub_b738

![Untitled](https://s2.loli.net/2022/02/11/j8BO6CbxMVthdSy.png)

hexdump打印一下sub_b738的内存，可以看到sub_b738的参数的值，每次执行算法时都会调用两次sub_b738函数 

![Untitled](https://s2.loli.net/2022/02/11/dDt1RmsvX2nFQT7.png)

第二次

![Untitled](https://s2.loli.net/2022/02/11/5VeKRdbH2oUh3XD.png)

把上面两次md5_update传递的字符串进行拼接再进行md5运算，结果跟第二次调用sub_b738时的第一个参数是一致的

![Untitled](https://s2.loli.net/2022/02/11/IRm2HQk93UxGq6E.png)

hook最上级函数sub_b9f4，在OnLeave的时候看下传出来的值

![Untitled](https://s2.loli.net/2022/02/11/1Tce3rpG9luvRE7.png)

hook这几个参数发现跟ida的反编译结果不太一样，根据打印的值猜测前面是字符串，后面是长度

![Untitled](https://s2.loli.net/2022/02/11/hNYUayKuIRsAEzD.png)

所以可以打印3组数据 

![Untitled](https://s2.loli.net/2022/02/11/VQLJuh1ti7w4Co2.png)

第一组数据是4e704c....

第二组数据是kanxue_imyang

第三组数据是4498220e，这个字符串跟加密的字符串是一致的

![Untitled](https://s2.loli.net/2022/02/11/XmuliQHvAbhItFR.png)

所以最后一个参数应该就是加密的字符串了

![Untitled](https://s2.loli.net/2022/02/11/kFKogC7WayQZrOR.png)

进入sub_b9f4函数可以看到result_buffer调用的地方

![Untitled](https://s2.loli.net/2022/02/11/y9vEBGsSfHdVihu.png)

接着hook sub_b738，此时对应的加密字符串是82900068....

进入sub_b738时

![Untitled](https://s2.loli.net/2022/02/11/fPXLdhk2BzH6ulA.png)

可以看到进入sub_b738函数前，有两次md5_update，第二次md5_update的值跟sub_b738第一个参数是一样的

![Untitled](https://s2.loli.net/2022/02/11/MTR7XH3SApqt2fK.png)

把这两次md5_update传递的字符串拼接到一起然后进入md5运算，得到的值就是最终的加密字符串了

![Untitled](https://s2.loli.net/2022/02/11/y8AHCUlsf6zeQL2.png)

在sub_b738函数内部又执行了两次md5_update

![Untitled](https://s2.loli.net/2022/02/11/8twiFBpXHQ5jOKA.png)

执行完sub_b738时，可以看到执行完时就已经是加密后的字符串了

![Untitled](https://s2.loli.net/2022/02/11/9Z7Ngn3wtxuFzyi.png)

两次md5_update然后执行sub_b738，这两次md5_update拼接的值是最终加密的字符串，在sub_b738内部又调用了两次md5_update，然后执行完毕，加密完成

其实源字符串通过hmac加密,key设置为kanxue_imyang结果跟上面是一样的，所以对于hmac加密只要找到key就可以了

![Untitled](https://s2.loli.net/2022/02/11/b7fc89HmtqTveKh.png)

HMAC算法内部对key进行了异或之类的处理，处理之后的值再进行md5_update，也就是说HMAC算法不是仅仅进行了md5_update盐字符串拼接源字符串再进行哈希，还会对md5_update盐字符串进行处理，然后再拼接源字符串，最后进行哈希

## 魔改异或值HMAC算法还原

hook sub_1470c，可以看到key，不过通过HMAC算法对源字符串进行加密得出的结果跟最终加密的结果不一致，说明算法进行了魔改，最终的加密结果放在最后一个参数中了

![Untitled](https://s2.loli.net/2022/02/11/Sw2gHYCRkz6AyF8.png)

对着参数按x，找到所有引用

![Untitled](https://s2.loli.net/2022/02/11/Xa5CxySQc1O7NFu.png)

先进入sub_11DF0查看，根据内部的特征很有可能就是md5_transform

![Untitled](https://s2.loli.net/2022/02/11/k3uYtUeOKPZpQDl.png)

它的上层函数就是md5_update

![Untitled](https://s2.loli.net/2022/02/11/hjdm2PACRZgXws7.png)

hook md5_update

![Untitled](https://s2.loli.net/2022/02/11/qRZMGh8F3u4UYkP.png)

可以看到md5_update的参数如下

![Untitled](https://s2.loli.net/2022/02/11/xeodSYA7OwpL4Ur.png)

没有进行魔改的HMAC算法md5_udpate的参数是这样的，有大量的0x36

![Untitled](https://s2.loli.net/2022/02/11/jtWeMVHB87pnfSq.png)

可以看到算法内部跟0x36异或的操作

![Untitled](https://s2.loli.net/2022/02/11/ey4TJPqNOKBGh3X.png)

跟0x36异或算出来的结果就是Key

![Untitled](https://s2.loli.net/2022/02/11/xIGlQucy3fOwKg4.png)

所以这个魔改的算法，修改了异或的值，把它跟0x88进行异或可以还原key

![Untitled](https://s2.loli.net/2022/02/11/hUINW4fcgqteES3.png)

进行算法还原也很容易，找到HMAC的实现代码，把异或的值修改一下就可以了 

## HMAC-SHA256算法还原

sub_14D68函数的参数传递了一个缓冲区

![Untitled](https://s2.loli.net/2022/02/11/rBd2RSuxLQ6kbz7.png)

打印看一下发现是kanxue_imgyang_52，因此这个函数应该就是核心算法函数 

![Untitled](https://s2.loli.net/2022/02/11/HQufiqovYgS1ZJF.png)

进入函数因为有混淆，所以跟踪第一个参数的调用情况，然后找到了一些常量，google后确认是sha256

![Untitled](https://s2.loli.net/2022/02/11/RdUceilgrypYk24.png)

根据源码可以确认sub_15194就是sha256_init函数

![Untitled](https://s2.loli.net/2022/02/11/VHYv5wQW6muOlLB.png)

回到上一层，可以确认sha256_init的参数来自于sub_15148的参数 

![Untitled](https://s2.loli.net/2022/02/11/hWMcSfZFIGdK7Dq.png)

最终可以确认sub_14d68的第一个参数是sha256_ctx

![Untitled](https://s2.loli.net/2022/02/11/NTFQOXquaADWhKl.png)

![Untitled](https://s2.loli.net/2022/02/11/cw6SloTr4KIG7py.png)

接着分析sub_9598，在里面找到常量，google一番找到标准算法搜索后看到是数组k里面的元素

![Untitled](https://s2.loli.net/2022/02/11/3Lv6H5VowycksKx.png)

看到k在sha256_transform内部进行了引用 

![Untitled](https://s2.loli.net/2022/02/11/1raQMfqB5d7pAEk.png)

因此这个函数就是sha256_transform

![Untitled](https://s2.loli.net/2022/02/11/mpORoeUc4dy7l3X.png)

查源码sha256_transform的上一层是sha256_update

![Untitled](https://s2.loli.net/2022/02/11/5yIsP34EvOeNJxH.png)

因此sub_9598的内部调用的是sha256_update

![Untitled](https://s2.loli.net/2022/02/11/uSXTOJ1kLChsxQV.png)

对sha256_update进行hook

![Untitled](https://s2.loli.net/2022/02/11/NQPO5ozZBLxgfeu.png)

对sub_15030进行hook

![Untitled](https://s2.loli.net/2022/02/11/tBclgqS3WjZI58M.png)

![Untitled](https://s2.loli.net/2022/02/11/6Hex9YQtTS3UuFa.png)

可以通过打印参数获取地址，可以分辨是不是同一个sha256_update

![Untitled](https://s2.loli.net/2022/02/11/XM1zP85dsq9KCGp.png)

两个md5_update拼接生成3b999....

![Untitled](https://s2.loli.net/2022/02/11/seSVZGIfkvmhObn.png)

再两次md5_update拼接生成最终加密字符串

![Untitled](https://s2.loli.net/2022/02/11/clpKr4tzX8Af2WZ.png)

![Untitled](https://s2.loli.net/2022/02/11/yPV9zpWo54CvwDF.png)

7=2$...字符串是key跟0x5c异或的结果，通过跟0x5c异或可以还原key

![Untitled](https://s2.loli.net/2022/02/11/ydLZEYqSHkrw79t.png)

通过key生成加密字符串

![Untitled](https://s2.loli.net/2022/02/11/1o9CqjtmZRIOYEH.png)