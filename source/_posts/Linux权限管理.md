---
title: Linux权限管理
toc: true
top: false
original: true
date: 2016-07-29 11:07:59
tags:
- Linux
categories:
- 学习笔记
---

Linux权限管理非常灵活，不像windows那一坨那么混乱。

现在来谈谈Linux下的权限管理吧。

```bash
vagrant@lucid32:/tmp$ ls -l /tmp/vagrant-shell
-(1)rwx(2)r-x(3)r-x(4) 1 vagrant(5) vagrant(6) 95 2016-07-29 04:49 /tmp/vagrant-shell
```

### 文件类型
首先来看第一部分，第一部分`-`表示文件类型，有以下几种文件类型：

符号|类型
:-:|:-:
-|普通文件
d|文件夹
l|符号链接
b|块设备文件
c|字符设备文件
s|套接文件
p|管道文件

### 文件权限
剩下的9个字符分为3组：分别是**归属用户**、**归属群组**、**其它用户**对于该文件的权限。`-`表示没有相应权限。

符号|权限
:-:|:-:
r|可读
w|可写
x|可执行

### 文件归属
第5、6部分分别代表归属用户、归属群组，也就是vagrant:vagrant。
其他部分不在讨论范围内。。

### 权限管理
重点来了，如果用户没有执行权限，那么他执行的时候会提示：
```bash
Permission denied
```
曾经的我多么幼稚，遇到这种情况就扔到`vFAT`分区执行= =

对于文件夹，必须拥有它的可执行权限，才能够使用`cd`命令进入该文件夹，我刚刚作死把`/usr/bin`的执行权限去掉了，和`rm -rf /usr/bin`效果雷同。。

拥有可读权限，才能够使用`ls`命令查看该文件夹的文件列表。

有了权限管理，就可以限制哪些人、哪些组能够访问文件了，而`root`拥有最高权限，可以无视一切= =

综上，应该对权限管理有个认识了，那么下面来说说怎么设置权限。

可以用3位**二进制**来表示权限，拥有什么权限那一位就为`1`，或者用一位**八进制**来表示，如下

权限|r|w|x
:-:|:-:|:-:|:-:
二进制|100|010|001
八进制|4|2|1

比如，`r-x`权限，可以表示成二进制的`101`，也可以表示成八进制的`5`。

前面说到，一个文件的权限有3组，分别为：**归属用户**、**归属组**、**其他用户**对应的权限，虽然用9位二进制来表示比较直观，但是用3位八进制来表示比较方便。

现在应该能明白常见的`755`权限了吧，其实就是：`rwxr-xr-x`权限。

### 设置权限
设置权限用`chmod`命令，比如，我想要一个文件只有我能访问，其他人都不能访问，那么我可以这样：
```bash
vagrant@lucid32:/tmp$ chmod 700 vagrant-shell
vagrant@lucid32:/tmp$ ls -l
total 4
-rwx------ 1 vagrant vagrant 95 2016-07-29 04:49 vagrant-shell
```

`chmod`还有几个常用的选项，如下
* a 所有用户  u 归属用户  g 归属群组  o 其它用户
* = 具有权限  + 增加权限  - 去除权限
* r 可读权限  w 可写权限  x 可执行权限 


那么上述的情况也可以这样写：
```bash
chmod go-rwx vagrant-shell
```
以上表示归属组、其他用户去掉`rwx`权限。
```bash
chmod a+rwx vagrant-shell
```
以上表示所有人都有`rwx`权限。

### SUID、SGID、Sticky bit
某些情况下，需要以可执行文件归属用户的身份执行该文件，可以为该文件设置`SUID`。同样，设置`SGID`能够以该文件归属群组的身份执行它。

例如：用户自行设定密码。出于安全方面的考虑，`/etc/shadow`只能由`root`用户直接修改。
`-rw------- root root /etc/shadow`

这个时候，可以为程序`/usr/bin/passwd`设置`SUID`，当普通用户执行`passwd`命令时，便能够以该程序归属用户`root`的身份修改`/etc/shadow`文件。而`passwd`程序自身带有身份验证机制，不能通过验证时拒绝执行，从而保证了安全。
```bash
ls -l /usr/bin/passwd
-r-s--x--x root root /usr/bin/passwd    
```

我们发现，归属用户的可执行权限位使用`s`，表示`SUID`。同样，归属群组的可执行权限位使用`s`，表示`SGID`。任何用户或群组都拥有 “其它用户” 的权限，所以不需要以其它用户身份执行文件，其它用户的可执行权限位便不会出现`s`。该权限位可能出现的属性为`t` ，也就是粘着位`Sticky bit`。
```bash
ls -ld /tmp
drwxrwxrwt root root /tmp    
```

粘着位表示任何用户都可能具有写权限，但只有该归属用户或`root`用户才能够删除。

`SUID`、`SGID`、`Sticky bit`也可以像权限一样，使用一个八进制数表示，如下：

* 4	SUID
* 2	SGID
* 1	Sticky bit
 
通过在`chmod`命令中使用4个八进制数的表达式，如`4755`，用第一位表示`SUID`、`SGID`或`Sticky bit`，便能够为文件设置这些特殊权限。示例：
```bash
chmod -R 4755 path
```

### 更改归属用户、归属组
更改归属用户、组可以用`chown`命令来实现，例如
```bash
vagrant@lucid32:/tmp$ sudo chown netcan:netcan vagrant-shell
vagrant@lucid32:/tmp$ ls -l
total 4
-rwxrwxrwx 1 netcan netcan 95 2016-07-29 04:49 vagrant-shell
```

### 参考链接
* [权限管理](http://i.linuxtoy.org/docs/guide/ch17s08.html)
