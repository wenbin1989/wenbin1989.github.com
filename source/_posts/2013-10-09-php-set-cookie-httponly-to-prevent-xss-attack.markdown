---
layout: post
title: "在php中设置cookie为httponly来减轻xss攻击的危害"
date: 2013-10-09 16:10
comments: true
categories: php
keywords: php cookie httponly xss PHPSESSIID
description: 
---

目前站点一般通过cookie保存登陆、认证等敏感信息，而通过xss窃取cookie，以实现窃取敏感信息、伪造用户登陆等，是一种常见的攻击方式。

防止xss的主要方法是对用户输入内容进行转义等等，可参见此雄文：
[Cross Site Scripting (XSS) Attacks: Methodology and Prevention](https://www.golemtechnologies.com/articles/prevent-xss)。
本文主要讨论在php下如何设置cookie为httponly，来最大限度的减轻xss攻击的危害，提升网站的安全性。

httponly是微软在Internet Explorer 6 SP1中为cookie引入的一个新特性，客户端脚本无法访问具有改属性的cookie值。
敏感cookie都应设置为httponly，减轻xss攻击的危害。

目前所有主流浏览器都支持该特性，php从5.2.0开始引入该特性。
只需在调用`setcookie`函数时，传入相关参数即可，如：
``` php
setcookie("key", $value, time()+60 * 60 * 24 * 10, "/", "", FALSE, TRUE);
```
具体可见[php文档](http://php.net/manual/en/function.setcookie.php)。

需要注意的是，销毁cookie时，`setcookie`后四个参数必须与设置cookie时严格相同，否则无法正确销毁。
例如销毁上面设置的cookie，需调用：
``` php
setcookie("key", "", time()-3600, "/", "", FALSE, TRUE);
```

还有一点特别需要注意，目前php的默认配置中，都会使用cookie来储存session id，见如下配置：
```
session.use_cookies = 1
session.use_only_cookies = 1
session.name = PHPSESSID
```
因此，有被截获cookie中的PHPSESSID，来劫持当前登陆用户会话的风险。
设置保存session id的PHPSESSID为httponly有两种方法。

1.  修改php.ini，设置`session.cookie_httponly = 1`；
2.  在调用`start_session()`之前调用`session_set_cookie_params()`，设置相关参数。如：
``` php
    session_set_cookie_params(0, "/", "", FALSE, TRUE);
```
具体可见[php文档](http://php.net/manual/en/function.session-set-cookie-params.php)。

两种方法各有优缺点。方法1必须改动php.ini，在某些虚拟主机无法实现，但方便快捷，修改一次全局生效。
方法2需要在每个`start_session()`调用，可能容易漏掉，但灵活性更强，也不用依赖配置环境。

等用户退出等需要主动销毁session的时候，除了`session_destroy()`以外，还需要销毁cookie中的PHPSESSID。

一般资料中介绍的方法如下
``` php
if (isset($_COOKIE[session_name()])) {
    setcookie(session_name(), '', time() - 3600);
}
```
但如前所述，如果销毁时`setcookie`参数跟创建时不同，将无法正确销毁，用改方法基本都会失败。
正确的做法为：
``` php
$sessionName = session_name();
$sessionCookie = session_get_cookie_params();

session_destroy();
$_SESSION = array();

if (isset($_COOKIE[$sessionName])) {
    setcookie($sessionName, false, $sessionCookie['lifetime'], $sessionCookie['path'], $sessionCookie['domain'], $sessionCookie['secure']);
}
```
