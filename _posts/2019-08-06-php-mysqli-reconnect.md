---
title: php连mysql实现断开重连
description: php连mysql实现断开重连
categories:
 - 开发常见问题解决
tags:
 - php
 - mysql
---

cron作业中循环通过 `mysqli` 调用数据库， 当数据库server端强制断开或者网络抖动， 和mysql服务端失联， 会导致客户端提示

```
<div style="border:1px solid #990000;padding-left:20px;margin:0 0 10px 0;">

<h4>A PHP Error was encountered</h4>

<p>Severity: Warning</p>
<p>Message:  mysqli_prepare(): MySQL server has gone away</p>
<p>Filename: service/mysql.php</p>
<p>Line Number: 784</p>
```

并且只有首次提示该报错， 且不会强制退出， 再次调用mysqli时， 只返回false。

通过 `mysqli_stmt_errno`和 `mysqli_stmt_error` 只返回

```
int(1)
string(30) "mysqli_prepare  error occurred"
```

错误原因，比较模糊不清， 无法定位到问题

通过mysqli的errno, 能准确获得错误码， 根据错误码， 我们按图索骥， 能查清到具体问题

```
current错误码是:2006, 这里的2006即是我们的具体错误码
current连接错误码是:0
```

经过排查， 错误码 `2006, 2013` 都是和服务端连接异常， 因此当我们检测到错误码是这两种时， 删除原有连接实例，进行重连来解决问题。


错误重现：

1. crontab作业中while循环，通过mysqli调用db查询
2. 从phpmyadmin中找到此次连接进程，`show full processlist` 手动查掉
3. 程序控制台中报错`MySQL server has gone away`
4. 再次跑循环1， 逻辑开始前，获取errorno为0， 但是mysqli_stmt_errno = 1， 仍然无法确认问题
5. 再次跑循环2，获取errno为2006， 即复现我们刚才说的问题。


所以针对上面问题拆解， 我们只需要在步骤5， 进行重连即可。

缺点是： 跑循环1 时，无法确认到问题， 即首次报错时， 未把报错信息写到mysqli实例中。

参考文章：

1. https://wiki.swoole.com/wiki/page/350.html
2. php-src-php-7.1.28/ext/mysqlnd/mysqlnd_enum_n_def.h
3. https://www.php.net/manual/zh/mysqli.errno.php
