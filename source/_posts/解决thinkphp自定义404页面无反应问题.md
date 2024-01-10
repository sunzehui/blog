---
title: 解决thinkphp自定义404页面无反应问题
tags:
  - php
abbrlink: 8c748b50
date: 2023-12-28 08:45:59
---

最近在做php，发现添加了自定义404页面模板后没反应，仍然使用默认异常模板，查了一下源码才知道问题所在，都是自己惯性思维惹的祸。

<!--more-->

文档说明：

[异常处理 · ThinkPHP5.1完全开发手册 · 看云 (kancloud.cn)](https://www.kancloud.cn/manual/thinkphp5_1/354092#HTTP__164)



```php
// \thinkphp\helper.php
if (!function_exists('abort')) {
    /**
     * 抛出HTTP异常
     * @param integer|Response      $code 状态码 或者 Response对象实例
     * @param string                $message 错误信息
     * @param array                 $header 参数
     */
    function abort($code, $message = null, $header = [])
    {
        if ($code instanceof Response) {
            throw new HttpResponseException($code);
        } else {
            throw new HttpException($code, $message, null, $header);
        }
    }
}
```

调用`thinkphp`的`abort`函数可以方便地创建HTTP异常，从这里入手，看一下`thinkphp`是怎么处理http异常的。

这里通过`$code`区分了一下，相当于是实现了函数重载，`$code`参数是404，走`HttpException`。

```php

class HttpException extends \RuntimeException
{
    private $statusCode;
    private $headers;

    public function __construct($statusCode, $message = null, \Exception $previous = null, array $headers = [], $code = 0)
    {
        $this->statusCode = $statusCode;
        $this->headers    = $headers;

        parent::__construct($message, $code, $previous);
    }
    // ...
}
```

初始化之后，会自动被`think\exception\Handle`处理，调用内部`render`方法，

```php
<?php
namespace think\exception;

class Handle
{
    /**
     * Render an exception into an HTTP response.
     *
     * @param  \Exception $e
     * @return Response
     */
    public function render(Exception $e)
    {
        if ($this->render && $this->render instanceof \Closure) {
            $result = call_user_func_array($this->render, [$e]);
            if ($result) {
                return $result;
            }
        }

        if ($e instanceof HttpException) {
            return $this->renderHttpException($e);
        } else {
            return $this->convertExceptionToResponse($e);
        }
    }

    /**
     * @param HttpException $e
     * @return Response
     */
    protected function renderHttpException(HttpException $e)
    {
        $status   = $e->getStatusCode();
        $template = Config::get('http_exception_template');
        if (!App::$debug && !empty($template[$status])) {
            return Response::create($template[$status], 'view', $status)->assign(['e' => $e]);
        } else {
            return $this->convertExceptionToResponse($e);
        }
    }

}
```

此时通过异常类型的判断，继续走`renderHttpException`，这个方法已经能够说明问题了，如果当前不是debug模式并且能够匹配到有对应模板，则会去找对应模板输出。显然我这里是走了else了，输出`App::$debug`值为`'1'`，但是我明明设置了`$debug=false`。继续查看`$debug`来源，发现初始化`$debug`是这样的：

```php
<?php
class App
{
    public static function initCommon()
    {
        // 初始化应用
        $config       = self::init();
        self::$suffix = $config['class_suffix'];

        // 应用调试模式
        self::$debug = Env::get('app_debug', Config::get('app_debug'));
		// ...
    }
}
```

先读取`env`，再读取`config.php`，哦原来是我`config.php`写死之后，`.env`文件没有删掉。

在`thinkphp`官方文档其实有写：

>环境变量中设置的`APP_DEBUG`和`APP_TRACE`参数会自动生效（优先于应用的配置文件），其它参数则必须通过`Env::get`方法才能读取。