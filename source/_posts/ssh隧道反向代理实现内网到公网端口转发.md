---
title: ssh隧道反向代理实现内网到公网端口转发
toc: true
top: false
original: true
date: 2016-09-28 18:04:39
tags:
- Linux
- 计算机网络
categories:
- 随笔
---

### 缘由
我写这篇文章，可以解决如下问题：在内网主机运行网站，无公网ip所以只能内网访问。但是主机内网可以访问外网，外网无法访问内网，现在我要做的就是通过反向代理，访问公网ip就能访问到内网主机网站，只需要一台`vps`。

以下我都会用`8080`端口来转发，因为我的`80`端口被占用了。

### 什么是代理
这里引用下知乎的答案。
> 代理其实就是一个中介，A和B本来可以直连，中间插入一个C，C就是中介。刚开始的时候，代理多数是帮助内网client访问外网server用的（比如HTTP代理），从内到外。后来出现了反向代理，"反向"这个词在这儿的意思其实是指方向相反，即代理将来自外网client的请求forward到内网server，从外到内。
> 作者：王明雨
> 链接：https://www.zhihu.com/question/24723688/answer/28828062
> 来源：知乎 
> 著作权归作者所有，转载请联系作者获得授权。

### 执行方案
首先`ssh`登录到**公网**主机，修改`sshd`的配置文件`/etc/ssh/sshd_config`，加入如下选项。
```bash
GatewayPorts yes
```

重启`sshd`，如下
```bash

# service sshd restart
```

这个选项的意思是，你通过反向代理的ip是公网ip（`0.0.0.0`），而不是loopback的ip（`127.0.0.1`）。后面会提到这个选项的具体作用。

然后`ssh`登录到**内网**主机，通过`ssh`来执行反向代理到**公网**主机，这里先给出几个参数的解释。
```bash
-N ssh不执行命令
-f 后台执行
-R 反向代理
```

所以只要执行以下命令，键入密码即可：
```bash
ssh -NfR 8080:localhost:80 <用户名>@公网主机ip
```

这里表示将**内网**主机的`80`的端口反向代理到**公网**主机的`8080`端口，如果你不开启`GatewayPorts`选项，就只能转发到**公网**主机的`127.0.0.1`网络上。

这时候在**公网**主机上执行：
```bash
$ netstat -tnl|grep 8080
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN
tcp6       0      0 :::8080                 :::*                    LISTEN
```
看到如上结果表明成功，这时候可以通过浏览器访问，**公网ip:8080**。

### 免密码、稳定隧道
上述的`ssh`方式不够稳定，而且每次断开重连都要键入密码，这里先给出免密码方式。

在**内网**主机上，执行
```bash
$ ssh-copy-id <用户名>@公网主机ip
```

以后通过这台**内网**主机`ssh`到**公网**主机，就不需要输入密码了。

至于稳定隧道，可以用`autossh`来解决，先安装`autossh`。
```bash

# apt-get install autossh
```

`autossh`的参数和`ssh`的参数一致，但是它会在隧道断开重连，省去了手动重连的方式。这里需要指出它的`-M`参数，`-M`参数指定一个端口用来做回显`echo`测试，它会在**公网**主机开一个端口专门接收**内网**主机的**心跳包**，然后回显回去（端口号+1），以表明隧道是否正常，若不正常则重连。

于是最终的命令如下：
```bash
autossh -M 23333 -NfR 8080:localhost:80 <用户名>@公网主机ip
```

为了开机自启动，可在`/etc/rc.local`写入上述命令。

### 最终方案
昨晚用`autossh`发现还是不稳定，`8080`端口会掉，而`-M`的端口不会掉，于是又换方案了，为`ssh`设置重连参数，修改**内网**主机的`~/.ssh/config`文件，增加如下参数：
```bash
ServerAliveInterval 60
ServerAliveCountMax 9999999999
```

* 第一个参数表示如果服务器（外网）没数据发来则过60秒客户端（内网）会发送一个空包到服务器，以保持tcp长连接，默认值为0，表示不会发心跳包，所以这里设置为60秒。
* 第二个参数表示，如果服务器（内网）没有收到心跳包指定次数，就中断连接。

今早起来，发现没掉，很稳定。


