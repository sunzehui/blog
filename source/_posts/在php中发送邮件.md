---
title: 在php中发送邮件
abbrlink: 17fa5d63
date: 2022-04-20 18:11:36
tags:
- php
- coding
---

客户网站纯静态的，现在想实现用户提交表单发送邮件给站长，因为空间环境是PHP的，所以准备用PHP实现一个发信程序，将信息传达到站长邮箱里。

<!--more-->



## 找轮子

看这个库比较火，10k star

[PHPMailer/PHPMailer: The classic email sending library for PHP (github.com)](https://github.com/PHPMailer/PHPMailer/)

文档上说要用composer，怪麻烦的，我直接把源码复制出来了，然后在自己的php文件引入。

文件结构

```
# ls -ll
total 9
drwxr-xr-x 1 Administrator 197121   0  4月 20 16:35 phpmailer/
-rw-r--r-- 1 Administrator 197121 610  4月 20 16:33 send.php
```



## 代码

在`send.php`中引入包，然后申请一下SMTP的授权码，我这边是QQ邮箱的。

```php
<?php
// send.php
require('./phpmailer/class.phpmailer.php');
require "./phpmailer/PHPMailerAutoload.php";


$phpmailer = new PHPMailer();

$phpmailer->IsSMTP();

$phpmailer->Host = 'smtp.qq.com';
$phpmailer->SMTPAuth = true;
$phpmailer->Username = 'h501574022@qq.com';
$phpmailer->Password = 'xxxxxxxxx';

$phpmailer->CharSet = 'utf-8';
$phpmailer->From = 'h501574022@qq.com';
$phpmailer->FromName = 'sunzehui';
$phpmailer->Subject = 'test';
$phpmailer->Body = 'haha';

$phpmailer->AddAddress('501574022@qq.com', 'Aseoe');

echo $phpmailer->send() ? '发送成功' : '发送失败';

?>
```

然后就收到邮箱了，任务完成

![image-20220420165850052](在php中发送邮件/image-20220420165850052.png)



但是这里php报了个warring

![image-20220420165943058](在php中发送邮件/image-20220420165943058.png)

大白话写着让我把`__autoload()`替换掉，使用`spl_autoload_register()`



然后我翻到源码是这样写的：

```php
// PHPMailerAutoload.php :46
if (version_compare(PHP_VERSION, '5.1.2', '>=')) {
    //SPL autoloading was introduced in PHP 5.1.2
    if (version_compare(PHP_VERSION, '5.3.0', '>=')) {
        spl_autoload_register('PHPMailerAutoload', true, true);
    } else {
        spl_autoload_register('PHPMailerAutoload');
    }
} else {
    /**
     * Fall back to traditional autoload for old PHP versions
     * @param string $classname The name of the class to load
     */
    function __autoload($classname)
    {
        PHPMailerAutoload($classname);
    }
}
```

明显是做了兼容性优化，在小于5.1.2时使用`__autoload()`，否则使用`spl_autoload_register()`。

我的版本是php7，至于为什么报这个错，我猜是因为php也会预编译，静态地检查代码块，而不管有没有可能走到这里，自认为这个warring设计的不好。



## 移植到客户网站

直接上效果图：

![1650448788593_](在php中发送邮件/1650448788593_.gif)

手动搞了个发送成功后跳转到首页的页面，很简陋。。



贴一下JavaScript代码：

```javascript
let index = 5
setInterval(() => {
    if (index === 0) {
        window.location.href = '/';
    }
    document.getElementById('t').innerHTML = index
    index--
}, 1000);
```

（其实我想做单页应用那样弹个窗提示成功，但是之前是使用的原生form表单，想想还是算了。）



对了，想在邮件内容里换行就要把消息设置一下，这样

```php
$phpmailer->CharSet = 'utf-8';
$phpmailer->From = 'h501574022@qq.com';
$phpmailer->FromName = '拓宝化工网站';
$phpmailer->Subject = '网站留言推送';
$phpmailer->Body = $content;
$phpmailer->IsHTML(true); // !!!在Body后面添加，不然不生效
```



参考：

[PHP测试成功的邮件发送案例 - 经验笔记 (cainiaojc.com)](https://www.cainiaojc.com/note/qacydj.html)

