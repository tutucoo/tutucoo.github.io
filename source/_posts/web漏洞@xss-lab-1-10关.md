---
title: web漏洞@xss-lab 1-10关
date: 2021-04-19 09:52:14
cover: https://gitee.com/tutucoo/images/raw/master/uPic/GURdwk.jpg
---

# xss-lab 1-10关

## 第一关 get参数存在xss

没啥好说的

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled.png)

## 第二关 输入框存在的xss

直接输入，无果

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%201.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%201.png)

不过可以看到插入到了value属性中，因此我们可以这样做，把前面的的双引号进行闭合

```php
" onclick=alert(1) ><"
```

最后点击输入框中触发 

## 第三关 输入框存在的xss(htmlspecialchars过滤)

正常插入，依旧没有弹框，检查源码

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%202.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%202.png)

看这造型跟上一关也是差不多的呀，于是" onclick=alert(1) ><"

不过很遗憾，不行

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%203.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%203.png)

看下源码

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%204.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%204.png)

首先接收输入框的输入赋值给keyword，然后生成一个form，里面的输入框的值由刚才输入的payload赋值，但是这里调用了htmlspecialchars，这个函数的作用如下：

- 将括号转换为 HTML 实体
- 将&号转换为HTML实体
- 将引号转换为HTML实体

因为我们输入的payload包含了大于号和小于号，所以不能使用这两个符号了

前面用双引号闭合依然不行，最后多了个双引号，为什么会出现这样的情况呢？

这需要了解一下单引号和双引号的区别:单引号中的内容不会进行解析，而双引号会进行解析，上图中我们可以发现，value的值是用单引号括起来的，因此里面的值会原封不动的显示出来

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%205.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%205.png)

所以，我们只需要通过单引号把前面的单引号封闭起来就可以了

```php
' onclick=alert(1)
```

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%206.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%206.png)

## 第四关 输入框存在的xss(htmlspecialchars过滤)

输入' onclick=alert(1)，没有弹，查看源码，发现无法跟value的引号进行闭合了，这是什么情况

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%207.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%207.png)

看下源码，看看是什么过滤先

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%208.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%208.png)

可以看到，首先将大小括号替换成了空，input控件的value的值是用双引号括起来的

这里有个规律，value的值如果最外层是双引号，则需要双引号去封闭，如果是单引号，就要用单引号去封闭，

使用下面的Payload

```php
" onclick=alert(1)
```

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%209.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%209.png)

## 第五关 过滤了关键字

输入一个基础的exp，可以看到\<script\>被过滤了

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2010.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2010.png)

使用大小写绕过依然不行，看下源码是怎么过滤的 

这里对<script和on进行了过滤

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2011.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2011.png)

这就容易了，不能用script和on就行了

```php
" > <iframe src=javascript:alert(1)></iframe>
```

iframe跟input不是一个控件，因此需要用>隔开

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2012.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2012.png)

## 第六关 大小写绕过

继续套用第五关用的exp，结果看到src被过滤

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2013.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2013.png)

然后试了几个，发现`<script><href>`也被过滤 ，看下源码，了解一下完整的过滤列表

可以看到以下几个过滤字段，但是并没有对大小括号进行过滤

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2014.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2014.png)

所以payload中，没这几个就可以了，这里用大小写绕过

```html
" ><SCRIPT>alert(1)</SCRIPT>
```

## 第七关 双写绕过

直接用上一关的exp，发现`<script>`被替换了

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2015.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2015.png)

看下源码有哪些过滤，这里在上一关的基础上增加了大小写过滤，直接将所有字符小写化，如果匹配上了就替换为空

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2016.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2016.png)

既然这样，我们就用双写绕过

```html
" oonnclick=alert(1)
```

此时后面还多了一个双引号

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2017.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2017.png)

用下面的exp进行封闭

```html
" oonnclick=alert(1) "
```

## 第八关 实体编码绕过

发现无法封闭

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2018.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2018.png)

看源码发现，不仅有之前的防护，还添加了对双引号的过滤，并且拒绝使用大小括号

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2019.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2019.png)

我们直接用`javascript:alert`也不行，因为中间的script会被替换

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2020.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2020.png)

使用下面的exp，t字符用`&#116`来代替

```html
javascript&#116:alert()
```

执行完看到友情链接中插入了代码

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2021.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2021.png)

点击友情链接，触发xss

## 第九关 输入合法性检查

还是使用上一关的exp，但是在前端代码中没有插入成功，看下源码进行了哪些防护

可以看到对链接进行了合法性判断，字符串前面必须是`http://`

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2022.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2022.png)

我们构造字符串把`http://`放在exp的前面肯定不行，这样会把js代码识别为网址

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2023.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2023.png)

exp如下，php中//表示注释，这样既可以绕过合法性检查，也可以把http://隔开

```html
javascrip&#116:alert()//http://
```

## 第十关 隐藏的按钮

这题没有看到输入框，在get参数中随意输入也没看到可注入的地方，查看前端代码可以看到默认有一些控件是隐藏的，我们把它显示出来 

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2024.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2024.png)

将hidden改为show，下图中的三个框显示出来了

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2025.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2025.png)

我们在get参数中添加t_sort

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2026.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2026.png)

在前端注入到value中

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2027.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2027.png)

最终的exp是:

```html
" onclick=alert() type="button"
```

exp中将type改为了按钮，这样就可以点击按钮进行触发了

![xss-lab%201-10%E5%85%B3%201d896838a3634be98810245205654d67/Untitled%2028.png](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2028.png)