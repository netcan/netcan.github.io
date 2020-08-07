title: 利用telnet在命令行发送/接收邮件
toc: true
original: true
date: 2016-02-20 10:09:42
tags:
- 计算机网络
categories:
- 随笔
---

明天就要去学校了。。有点不舍，本来今天就要去的，因为玉林到合肥有票了，就改签到明天，因为学生票还省下了100。。。

废话不多说，这几天在学《计算机网络》，发现`telnet`其实是个很万能的东西，我曾对它的印象仅仅停留在远程控制，那时候在机房叫老师打开机房的网络，老师在`cmd`就用`telnet`远程控制来打开网络。

比如用`telnet`发送邮件，就可以深刻理解协议的含义：**语义**、**语法**和**时序**，以后`socket`编程的时候就知道怎么设计协议了，其实和设计指令差不多。

因为*QQ邮箱*加密传输了，不好测试，这里用*139移动邮箱*发送到*QQ邮箱*来测试。
<!--more-->

注意退出`telnet`的话，先按`CTRL+J`，然后输入`quit`即可退出。

`base64`加密的话：

	echo -n 待加密的内容 | base64

先看看发送图(`smtp`)：
![发邮件](http://7xibui.com1.z0.glb.clouddn.com/%40%2F2016%2F02%2Fscreenshot-area-2016-02-20-103534.png)

再来看看收邮件的(`pop3`)：

	✘  ~/WorkSpace/Netcan_Web/blog/source/Timeline  telnet pop.139.com 110
	Trying 221.176.9.169...
	Connected to pop.mail.10086.cn.
	Escape character is '^]'.
	+OK richmail system v10(32d256c7d202533-1a2ff)
	USER netcan
	+OK
	PASS xxxxxx
	+OK login success
	LIST
	+OK
	1 11481
	2 20476
	3 22995
	4 17824
	5 16520
	6 48992
	7 50115
	8 51141
	9 50336
	10 18149
	11 51062
	12 12116
	13 20278
	14 26734
	15 51260
	16 21147
	17 51499
	18 11584
	19 20796
	20 12594
	21 52576
	22 16798
	.
	RETR 1
	+OK
	Received:from mail139@139.com (unknown[172.16.202.6])
		by rmoda-17132 (RichMail) with ODA id 42ec5409b9d04b8-050cc;
		Fri, 05 Sep 2014 21:25:36 +0800 (CST)
	X-RM-TRANSID:42ec5409b9d04b8-050cc
	From: =?gb2312?B?1tC5+tLGtq8xMznTys/k?= <mail139@139.com>
	Message-ID: <744425079.27371409923191498.JavaMail.hadoop@q201-o06-9>
	Subject: =?GB2312?B?1tC5+tLGtq/M4dDRo7rH68T6udjXorG+u/q6xQ==?=
	=?GB2312?B?wuu1scewu7C30dPgtu6jrM/qx+nH67Xju/ey6b+0?=
	MIME-Version: 1.0
	Content-Type: multipart/alternative;
	boundary="----=_Part_2737_1393666244.1409923191496"
	X-RICHINFO:CATEGORY:4326;PRODUCTOPERATIONMAIL:2001;SELFCLOSED:1;M:;TEMPLATELIST:2_4;MAILORDERID:141956;NOTIFYTYPE:14;ISMULTITEMPLATE:1;BUSSINESSID:1443;MJ:0;ISWAPNOTIFY:0;CHANNEL:30
	X-TDZXINFO: ODACHANNEL:30;ODACATEGORY:4326
	X-OP_RICHINFO_MAILTYPE:0
	Date:
	To:netcan

	------=_Part_2737_1393666244.1409923191496
	Content-Type: text/plain;charset="gbk"
	Content-Transfer-Encoding: base64

	oaHX8L60tcS/zbuno7pbsunT4LbuXWh0dHA6Ly93YXBtYWlsLjEwMDg2LmNuL2ahoQqhoczh0NHE
	+qO6yta7+sno1sNDTVdBUL3TyOuju8H3wb+wtNbQufrSxravR1BSU7Hq17y8xsvjoaMKCQkJCQkJ
	CQkKCQkJCQkJCQkKCQkJCQkJCQkKCQkJCQkJCQkKCQkJCQkJCQkKCQkJCQkJICAJIAkgCgkJCgkJ
	CQkJCQkJCgkJCQkJCgkJCQkJCQkJCiAJCQkJCQkgICAJIAogICAgICAgICAJCQo=
	------=_Part_2737_1393666244.1409923191496
	Content-Type: text/html;charset="gbk"
	Content-Transfer-Encoding: base64

	PGh0bWw+PGhlYWQ+PHRpdGxlPqGhPC90aXRsZT48bWV0YSBodHRwLWVxdWl2PSJDb250ZW50LVR5
	cGUiIGNvbnRlbnQ9InRleHQvaHRtbDsgY2hhcnNldD1nYjIzMTIiLz48L2hlYWQ+PGJvZHkgYmdj
	b2xvcj0iI0ZGRkZGRiIgbGVmdG1hcmdpbj0iMCIgdG9wbWFyZ2luPSIwIiBtYXJnaW53aWR0aD0i
	MCIgbWFyZ2luaGVpZ2h0PSIwIj48c3BhbiBzdHlsZT0iIGZvbnQ6MHB4LzBweCAny87M5Sc7IGRp
	c3BsYXk6bm9uZTsiPtfwvrS1xL/Nu6ejuluy6dPgtu5dPGEgaHJlZj0iaHR0cDovL3dhcG1haWwu
	MTAwODYuY24vZiIgdGFyZ2V0PSJfYmxhbmsiPmh0dHA6Ly93YXBtYWlsLjEwMDg2LmNuL2ahoTwv
	YT48YnIvPqGhzOHQ0cT6o7rK1rv6yejWw0NNV0FQvdPI66O7wffBv7C01tC5+tLGtq9HUFJTserX
	vLzGy+Ohozxici8+PC9zcGFuPjx0YWJsZSB3aWR0aD0iNzgwIiBib3JkZXI9IjAiIGFsaWduPSJj
	ZW50ZXIiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgaWQ9Il9fMDEiIHN0eWxlPSJi
	b3JkZXItY29sbGFwc2U6Y29sbGFwc2U7IHdpZHRoOjc4MHB4OyBib3JkZXItc3BhY2luZzowO3Bh
	ZGRpbmc6MDsgbWFyZ2luOjAgYXV0bzsiPjx0cj48dGQ+CQkJPGltZyBzcmM9Imh0dHA6Ly9mdW4u
	bWFpbC4xMDA4Ni5jbi9jbi8xNDA3LzA4MjIvaW1hZ2VzL2luZGV4XzAxLmpwZyIgd2lkdGg9Ijc4
	MCIgaGVpZ2h0PSI4OCIgYWx0PSIiIGJvcmRlcj0iMCIgc3R5bGU9ImRpc3BsYXk6YmxvY2s7Ii8+
	PC90ZD4JPC90cj48dHI+PHRkPgkJCTxpbWcgc3JjPSJodHRwOi8vZnVuLm1haWwuMTAwODYuY24v
	Y24vMTQwNy8wODIyL2ltYWdlcy9pbmRleF8wMi5qcGciIHdpZHRoPSI3ODAiIGhlaWdodD0iMTA3
	IiBhbHQ9IiIgYm9yZGVyPSIwIiBzdHlsZT0iZGlzcGxheTpibG9jazsiLz48L3RkPgk8L3RyPjx0
	cj48dGQ+CQkJPGltZyBzcmM9Imh0dHA6Ly9mdW4ubWFpbC4xMDA4Ni5jbi9jbi8xNDA3LzA4MjIv
	aW1hZ2VzL2luZGV4XzAzLmpwZyIgd2lkdGg9Ijc4MCIgaGVpZ2h0PSI4MiIgYWx0PSIiIGJvcmRl
	cj0iMCIgc3R5bGU9ImRpc3BsYXk6YmxvY2s7Ii8+PC90ZD4JPC90cj48dHI+PHRkPgkJCTxpbWcg

	------=_Part_2737_1393666244.1409923191496--
	.

下面是步骤：

## 发邮件
1. 使用`telnet SMTP地址 端口`连接
2. 发送`HELO 邮箱服务器名称`
3. 发送`AUTH LOGIN`进行登陆，第一次输入的是`base64`加密过的用户名，第二次是`base64`加密过的密码。
4. 发送`MAIL FROM: <xxx@xx.com>`准备发邮件
5. 发送`RCPT TO: <xx@xx.com>`指定目标邮箱地址
6. 发送`DATA`开始输入邮件内容，一行用`.`结束。
7. 发送`QUIT`退出

## 收邮件
1. 使用`telnet POP3地址 端口`连接
2. 发送`USER 用户名`进行登陆
3. 发送`PASS 密码`确定登陆
4. 发送`LIST`查看邮件列表
5. 发送`RETR 编号`查看邮件信息
6. 发送`DELE 编号`删除邮件
7. 发送`QUIT`退出

