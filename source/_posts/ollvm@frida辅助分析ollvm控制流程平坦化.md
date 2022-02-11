# frida辅助分析ollvm控制流程平坦化

本例调用了sign2函数，传递了2个参数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled.png](https://s2.loli.net/2022/02/11/h4ycGRo1V8E7JB6.png)

平坦化的流程图 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%201.png](https://s2.loli.net/2022/02/11/g3918dMOTNhWD7H.png)

像这种有大量while循环的一般就是控制流平坦化了 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%202.png](https://s2.loli.net/2022/02/11/L8tVA9qha2SQIpG.png)

hook jni函数 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%203.png](https://s2.loli.net/2022/02/11/1hKE5zHndemxvAN.png)

每次都会传递随机的两个参数，为了方便分析，需要每次都传递相同的值，这就需要主动调用了

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%204.png](https://s2.loli.net/2022/02/11/si4b9PjlBRODnXd.png)

发现每次传入一样的参数，返回值还是会变化，说明jni函数里面还有其他的随机操作，但是返回值有时候会重复，说明不是完全随机的

![Untitled](https://s2.loli.net/2022/02/11/oamlOVZSnucG2Ci.png)

IDA分析JNI代码，因为OLLVM混淆过了，很多无意义的代码，就无法通过f5进行分析了，这时需要查看参数的引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%206.png](https://s2.loli.net/2022/02/11/Y5uWUISKpihrtOm.png)

把v5改为_str_1，再次查看_str_1的引用，发现两处引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%207.png](https://s2.loli.net/2022/02/11/bXOSkJorUlvhI5M.png)

看第二处引用，ReleaseStringUTFChars第一个参数是JNIEnv*，第二个参数是要释放的字符串，第三个参数是通过调用GetStringUTFChars(_str1_1)得到的，因为这里_str_1被释放了，但是v9还有引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%208.png](https://s2.loli.net/2022/02/11/isaqIVKX1uZtbeW.png)

v9的引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%209.png](https://s2.loli.net/2022/02/11/Anby6B259xXsaqH.png)

这是v9所有引用的地方，主要看sub_13558，v45应该是传出来的值，V9是传入的字符串，V10是长度

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2010.png](https://s2.loli.net/2022/02/11/A5KzF6GTPNcJdmR.png)

接着看v45的引用，如果引用比较多，优先看函数调用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2011.png](https://s2.loli.net/2022/02/11/BgGXTZoz4PC5Qxq.png)

进入sub_13558函数，混淆比较严重

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2012.png](https://s2.loli.net/2022/02/11/6vkQG8JB9WIM3Lp.png)

因为混淆比较严重，所以对它进行hook，看下调用时传入的字符串以及返回时传出的字符串，进入的时候打印的参数是str1，返回的时候打印的是v45

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2013.png](https://s2.loli.net/2022/02/11/xWbzRQtUO6ywTI3.png)

每次主动调用时，sub_13558都会被调用两次，第一次传入0123456789，第二次传入abcdefgh

![Untitled](https://s2.loli.net/2022/02/11/3ohIiJFAU1aR9nz.png)

给v45重新命名为str_1_str

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2015.png](https://s2.loli.net/2022/02/11/x4uMXGPwveWJySU.png)

然后再看sub_12d70，这时可以发现str_1_str被得重新命名为v44了，sub_13558的第2个参数变成v16了，本来是v9的

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2016.png](https://s2.loli.net/2022/02/11/XCq52dpj9MQ8Lun.png)

v16来自v15，v15又来自v6

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2017.png](https://s2.loli.net/2022/02/11/TQzpoe5a8fBqkDX.png)

再看v6引用，发现是str_2，所以str_2也会传入sub_13558

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2018.png](https://s2.loli.net/2022/02/11/sbEitO8X9CS3YBN.png)

原来的str_1_str重命名为str_2_str

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2019.png](https://s2.loli.net/2022/02/11/2IW1957CVJMlEZU.png)

再看sub_12d70，参数命名也产生了变化

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2020.png](https://s2.loli.net/2022/02/11/ZVNt7OTnUcxwp8Q.png)

hook sub_12d70看一下，进入的时候打印str_1_str和str_2_str，执行完之后再次打印

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2021.png](https://s2.loli.net/2022/02/11/r4HuctDWnLYPomX.png)

sub_12d70执行前和执行后的参数的值，可以看到执行后没有产生变化

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2022.png](https://s2.loli.net/2022/02/11/CEm24i6ngjtV18o.png)

IDA中的sub_12D70的参数解析莫明奇妙发生了变化，看下v50的引用，可以看到v50的引用都是大于124行的，也就是说在124行之前是没有进行赋值的，那么有可能就是在sub_12d70函数内部进行赋值的

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2023.png](https://s2.loli.net/2022/02/11/jXclHTpC3YDhkAR.png)

看下v50的值是什么

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2024.png](https://s2.loli.net/2022/02/11/4DwtYbremLvunNl.png)

v50的值如下图所示，好像也看不出个啥

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2025.png](https://s2.loli.net/2022/02/11/3qB8wcGvYJAU5iT.png)

再看下sub_12d70有没有返回值，x86通常将返回值保存在eax寄存器里面，arm32通常保存在r0，arm64通常把返回值保存在w0或x0，不过这里并没有看到返回值  

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2026.png](https://s2.loli.net/2022/02/11/IQByqvoXPxtfT3k.png)

再经过一番分析，发现没有其他有引用的地方，而且对v50进行了操作，如果没有值是不会这么操作的，所以再看v50，看之前是不是打印错了 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2027.png](https://s2.loli.net/2022/02/11/4B8HrN6b9CzajTM.png)

v50赋值给了v42，再看sub_13558调用时传递了v42，这个sub_13558就是一开始碰到的函数

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2028.png](https://s2.loli.net/2022/02/11/dWDoTZKSzU4hGOX.png)

看sub_13558的返回值是v28，继续往下看sub_162b8，传递了v28

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2029.png](https://s2.loli.net/2022/02/11/zwCyxfSYJAvOqdG.png)

hook sub_13558，看下返回值 v28

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2030.png](https://s2.loli.net/2022/02/11/jpHQSwCKeoIlvRJ.png)

可以看到v28的值是个字符串

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2031.png](https://s2.loli.net/2022/02/11/2JEkKp3N5Tjrq1w.png)

hook sub_162b8

```jsx
var sub_162B8 = base_hello_jni.add(0x162B8);
Interceptor.attach(sub_162B8, {
    onEnter: function(args){
        console.log("sub_162B8 onEnter:",args[0],args[1],args[2]);
    },onLeave: function (retval) {
        console.log("sub_162B8 onLeave:",retval);
    }
});
```

可以看到传递的三个参数以及返回值，这几个值看起来像指针和长度

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2032.png](https://s2.loli.net/2022/02/11/Ga369DFtpCgOhN7.png)

dump看一下 

```jsx
var sub_162B8 = base_hello_jni.add(0x162B8);
Interceptor.attach(sub_162B8, {
    onEnter: function(args){
        console.log("sub_162B8 onEnter:",hexdump(args[0]),"\r\n",args[1],args[2]);
    },onLeave: function (retval) {
        console.log("sub_162B8 onLeave:",hexdump(retval));
    }
});
```

第一个参数可以看出，是将传入的str1和str2拼接起来了，第三个参数是空的，因此可以猜测它是返回值

![Untitled](https://s2.loli.net/2022/02/11/CjtPIHu6ULlFMis.png)

函数返回时打印一下第三个参数

```jsx
var sub_162B8 = base_hello_jni.add(0x162B8);
Interceptor.attach(sub_162B8, {
    onEnter: function(args){
        console.log("sub_162B8 onEnter:",hexdump(args[0]),"\r\n",args[1],args[2]);
    },onLeave: function (retval) {
        console.log("sub_162B8 onLeave:",hexdump(args[2]),hexdump(retval));
    }
});
```

第三个参数的值是19，它是返回值的长度

![Untitled](https://s2.loli.net/2022/02/11/KRyBXkSQUv2oZNq.png)

sub_162b8重新命名

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2035.png](https://s2.loli.net/2022/02/11/icrCPnbWhSmJ7Xa.png)

进入sub_162b8进行分析

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2036.png](https://s2.loli.net/2022/02/11/u5XzcHKafnGgrJe.png)

input_buffer给v13和v14用了 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2037.png](https://s2.loli.net/2022/02/11/kMdo6GCYFJX4BTt.png)

对v14重命名

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2038.png](https://s2.loli.net/2022/02/11/ITUtQLydYfm3rDF.png)

再找_input_buffer引用，发现v21用到了 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2039.png](https://s2.loli.net/2022/02/11/XELO9ycaHiR7GpQ.png)

v21给了v40

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2040.png](https://s2.loli.net/2022/02/11/Fhx1WvLsTGijk4g.png)

然后v40又给了v41

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2041.png](https://s2.loli.net/2022/02/11/oTyYAdtxkevLOhU.png)

这里引用了一个字符数组，这个一般是base64编码 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2042.png](https://s2.loli.net/2022/02/11/wOHlLN3pZgiyVIt.png)

这里把返回值Mdey那个字符串直接在010中转码

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2043.png](https://s2.loli.net/2022/02/11/6baOeqHzkrtmgTX.png)

结果就是传入的str1和str2，所以可以确认sub_162b8是base64算法

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2044.png](https://s2.loli.net/2022/02/11/WiJhUv2CgcBoFp9.png)

接着看经过base64加密之后的后续加密

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2045.png](https://s2.loli.net/2022/02/11/yT7z6N9hnUgIJa3.png)

先看sub_130f0

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2046.png](https://s2.loli.net/2022/02/11/usv2fiYPGh7Sw8o.png)

进入sub_130f0 f5再返回发现函数的参数变了，这些多出来的参数先不用管它 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2047.png](https://s2.loli.net/2022/02/11/pnf3i7xRrNBDVZ2.png)

hook sub_130f0

```jsx
var sub_130f0 = base_hello_jni.add(0x130f0);
Interceptor.attach(sub_130f0, {
    onEnter: function(args){
        this.arg0 = args[0];
    },onLeave: function (retval) {
        console.log("sub_130f0 onLeave:",hexdump(this.arg0));
    }
});
```

函数执行之后第一个参数的值，该函数调用了多次

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2048.png](https://s2.loli.net/2022/02/11/kxfrdT1iUs6LR8M.png)

![Untitled](https://s2.loli.net/2022/02/11/zWsHbqIduOtKG8x.png)

再看sub_15f1c

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2050.png](https://s2.loli.net/2022/02/11/KNOemq5XD3EFbk7.png)

v37引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2051.png](https://s2.loli.net/2022/02/11/yUbXpd4YSB8lorP.png)

先hook sub_15f1c，着重看下函数执行完之后第三个参数的值 

```jsx
var sub_15F1C = base_hello_jni.add(0x15F1C);
Interceptor.attach(sub_15F1C, {
    onEnter: function(args){
        this.arg2 = args[2];
        console.log("sub_15F1C onEnter:",args[0],args[1],this.arg2);
    },onLeave: function (retval) {
        console.log("sub_15F1C onLeave:",this.arg2);
    }
});
```

三个参数的值

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2052.png](https://s2.loli.net/2022/02/11/tpRVaBGKwuTMhHv.png)

函数执行后第三个参数的值，没有发生变化

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2053.png](https://s2.loli.net/2022/02/11/4tO7yCmnf3lgqes.png)

第一个参数是个指针，修改脚本，改变打印的方式

```jsx
var sub_15F1C = base_hello_jni.add(0x15F1C);
Interceptor.attach(sub_15F1C, {
    onEnter: function(args){
        this.arg2 = args[2];
        console.log("sub_15F1C onEnter:",ptr(args[0]).readCString(),this.arg2);
    },onLeave: function (retval) {
        console.log("sub_15F1C onLeave:",hexdump(this.arg2));
    }
});
```

第一个参数是之前base64编码字符串，第二个参数是长度，第三个参数是个指针

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2054.png](https://s2.loli.net/2022/02/11/oJN6A4SjPZmDYl5.png)

函数执行完之后，第三个参数的内容是03b5....，然后看sign2的加密结果也是03b5...

![Untitled](https://s2.loli.net/2022/02/11/reUkKPo7nLmjwHv.png)

进入sub_15f1c

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2056.png](https://s2.loli.net/2022/02/11/iJKDMubdkIYzVOZ.png)

再进入sub_15a28

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2057.png](https://s2.loli.net/2022/02/11/GaoU7J9PvqxRSrK.png)

buffer赋值给了v4

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2058.png](https://s2.loli.net/2022/02/11/ZyqdOR3HBz5tQws.png)

再看v4引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2059.png](https://s2.loli.net/2022/02/11/ikGfXM5tjzubseD.png)

看下sub_154d4函数，第二个参数是buffer，第三个参数是len

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2060.png](https://s2.loli.net/2022/02/11/fU2PcAo8dqObaDj.png)

hook sub_154d4看下第一个参数

```jsx
var sub_154D4 = base_hello_jni.add(0x154D4);
Interceptor.attach(sub_154D4, {
    onEnter: function(args){
        this.arg0 = args[0];
        console.log("sub_154D4 onEnter:",ptr(args[1]).readCString());
    },onLeave: function (retval) {
        console.log("sub_154D4 onLeave:",hexdump(this.arg0));
    }
});
```

看结果不是这个函数返回了最终的加密字符串

![Untitled](https://s2.loli.net/2022/02/11/uWmxEKdbaFMec9p.png)

接着看out_buffer的引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2062.png](https://s2.loli.net/2022/02/11/9OgzB6fhECGx1v7.png)

将v5改名为_out_buffer

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2063.png](https://s2.loli.net/2022/02/11/JzVKi8bTNsxq9pI.png)

再看_out_buffer的引用

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2064.png](https://s2.loli.net/2022/02/11/4TDU9CYr7l2vbPd.png)

hook sub_158ac

```jsx
var sub_158AC = base_hello_jni.add(0x158AC);
Interceptor.attach(sub_158AC, {
    onEnter: function(args){
        this.arg1 = args[1];
    },onLeave: function (retval) {
        console.log("sub_158AC onLeave:",hexdump(this.arg1));
    }
});
```

函数执行之后，输出跟最终的加密字符串一致

![Untitled](https://s2.loli.net/2022/02/11/JlBSgPo8TrkdDZh.png)

![Untitled](https://s2.loli.net/2022/02/11/GCiyARFEaHIYSWf.png)

进入sub_158ac，这个函数比较简单只有一个函数sub_154d4

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2067.png](https://s2.loli.net/2022/02/11/osOreY7udIKD6UW.png)

查看参数引用，out_buffer赋值给了v9

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2068.png](https://s2.loli.net/2022/02/11/BUxbNZnywvtFlKW.png)

v9重命名为_out_buffer，再看它的引用，v12的值给了_out_buffer

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2069.png](https://s2.loli.net/2022/02/11/WdQnZUCDLwTuy4k.png)

v12的值又是a1给的，下面又调用了sub_154d4

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2070.png](https://s2.loli.net/2022/02/11/m6fFuXKGoY4HMPU.png)

进入sub_154d4，查看第一个参数result的引用，sub_14844的返回值给了result

![Untitled](https://s2.loli.net/2022/02/11/2wRT9rPZ3gIcVGp.png)

看下sub_14844，这个函数内部有很多常量

3614090360转为十六进制，再百度一下，可以看到是md5

![Untitled](https://s2.loli.net/2022/02/11/KB1vYRWNJMQs4z2.png)

如果常量只是用来跳转或判断一般就不是加密算法 

![frida%E8%BE%85%E5%8A%A9%E5%88%86%E6%9E%90ollvm%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%B9%B3%E5%9D%A6%E5%8C%96%20f42399ba539040ec9909d01879e4c5fc/Untitled%2073.png](https://s2.loli.net/2022/02/11/cGLrRSXxUAePNdD.png)

根据最终字符串是个32位的字符串，可以猜测是个md5

再打印一个sub_154d4

总结一下算法：

1、调用sign2函数，传递0123456789和abcdefghakdjshgkjadsfhgkjadshfg

2、拼接成0123456789abcdefghakdjshgkjadsfhgkjadshfg

3、对0123456789abcdefghakdjshgkjadsfhgkjadshfg进行base64编码

4、加盐拼接（加盐字符串+++++++++salt2+MDEy.....)

5、进行md5运算

逆向的核心是通过输入参数找到算法加密的过程，或从输出结果回溯加密过程

![Untitled](https://s2.loli.net/2022/02/11/wBHErj3a29lkC6M.png)

计算出的结果跟最终加密的结果一样（注意这里输入最后要回车，不然结果不对）

![Untitled](https://s2.loli.net/2022/02/11/B96xifZjLQb8XSh.png)