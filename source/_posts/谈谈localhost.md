---
title: 谈谈localhost
toc: false
top: false
original: true
date: 2016-12-25 21:58:07
tags:
- 计算机网络
categories:
- 随笔
---

刚刚发现一个有趣的现象，测试`django`项目的时候，这样启动服务器：

```bash
$ python3 ./manage.py runserver
...
Starting development server at http://127.0.0.1:8000/
...
```

然后想起`laravel`项目也是`8000`端口，就同时启动`laravel`项目：
```bash
$ php artisan serve
Laravel development server started on http://localhost:8000/
```

问题来了，2个端口一样，按理说应该会冲突啊，可是没有，打开浏览器，输入`127.0.0.1:8000`，看到的是`django`的项目。那么`laravel`项目哪去了，难道没启动成功，可是又没报错？

仔细看`laravel`的输出信息，于是浏览器输入`localhost:8000`，果然，是`laravel`的项目，也就是说，127.0.0.1 != localhost？这不可能吧。

于是我用`netstat`看一下，到底是什么情况。

```bash
$ netstat -apn | grep 8000
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      5374/python3
tcp6       0      0 ::1:8000                :::*                    LISTEN      5518/php7.0
```

从输出信息看，确实都是`8000`端口，但是地址没发生冲突，也就是说`django`项目默认是用`ipv4`的`127.0.0.1`地址，`laravel`项目默认用的是`ipv6`地址`::1`，地址不一样，那么端口相同也可以。

然后我关掉这2个项目，单独启动`laravel`项目，发现默认用的就是`ipv6`地址，也就是说不能用`127.0.0.1`来访问，可以考虑用这2个地址访问：
- http://localhost:8000/
- http://[::1]:8000/

看了下`wiki`上的[Localhost](https://en.wikipedia.org/wiki/Localhost)介绍，操作系统的`hosts`文件中的`localhost`解析一些`ip`：

```
127.0.0.1    localhost
::1          localhost
```

那么问题来了，解析`localhost`是不是有个优先级问题呢？我发现当`laravel`项目启动的时候，默认是解析成`ipv6`的地址`::1`，而**只**开启`django`项目的时候，才会解析成`ipv4`的地址`127.0.0.1`。

考虑到其他框架可能会占用`8000`端口，那么有可能`laravel`采用了默认利用`ipv6`地址，从而避免启动端口冲突问题；然而`localhost`先解析`ipv6`，从而避免访问冲突问题，真是机智。。。

所以需要用这种方式对`laravel`项目做端口映射就要注意了，用`--hosts`参数指定`ip`，例如：

```bash
$ php artisan serve --host=127.0.0.1
```

那么`dns`是否会优先解析`ipv6`呢？我猜应该是的，因为`ipv6`正在慢慢普及。
