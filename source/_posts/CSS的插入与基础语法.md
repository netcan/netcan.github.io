title: CSS的插入与基础语法
toc: true
original: true
tags:
  - CSS
categories:
  - 学习笔记
date: 2015-03-24 15:15:22
updated: 2015-03-24 15:15:22
---

### css样式表
使用方式包括行内样式、内嵌样式、链接样式、导入样式。

#### 行内样式
直接在html标记中使用style属性，属性的内容就是css的属性和值。

	<p style="color:red">段落样式</p>

#### 内嵌样式
添加css样式代码到<head></head>之间，用<style></style>标记声明。

	<!DOCTYPE HTML>
	<html>
	<head>
	<meta charset="utf-8">
	<title>内嵌样式</title>
	<style type="text/css">
		p {
			color: red;
			font-size: 12px;
		}
		</style>
	</head>
	<body>
	<p>采得百花成蜜后</p>
	<p>为谁辛苦为谁甜</p>
	</body>
	</html>
效果：
{% iframe http://7xibui.com1.z0.glb.clouddn.com/uploads/web_dev/css_study/1_1_2.html %}



#### 链接样式
在<head></head>标记中采用<link>标签，

	<link rel="stylesheet" type="text/css" href="file.css" />

#### 导入样式
在<style></style>标记中，使用@import导入一个外部样式表

	<style type="text/css">
		@import "file.css"
	</style>

### 优先级问题
浏览器在显示网页是这样处理的，先检查有没有行内样式，有就执行，针对本句的其他css就不去管了，其次检查内嵌样式，有就执行，若前两者都没有的情况下再检查链接样式的css。综上所述，3种css的执行优先级是：行内样式、内嵌样式、链接样式。

### css选择器
按用途可以分为标签选择器、类选择器、全局选择器、ID选择器、伪类选择器等。

#### 标签选择器
基本形式如下所示

	tagName { property: value }
其中tagName表示标记名称，比如p,h1等html标记，property表示css属性，value表示属性值。

	body, p, td, th, div, blockquote, dl, ul, ol {
		font-family: Tahoma;
		font-size: 1em;
		color:  #000000;
		}

#### 类选择器
语法格式如下所示

	.classValue { property: value }
其中classValue是选择器的名称，自己命名，如果一个标记中具有class属性且class属性值为classValue，那么呈现的样式由该选择器指定。在定义类选择器时要加句点.

#### ID选择器
语法格式如下所示

	#idValue { property: value }
ID选择器定义的是某一个特定的html元素，一个网页文件只能有一个元素使用某一ID的属性值。

#### 全局选择器
语法格式如下所示

	* { property: value }
*表示对所有元素起作用。

#### 组合选择器
只是一种组合形式，并不是一种真正的选择器，但在实际中经常使用，例如

	h1.red {color:red}
	<h1 class="red">h1</h1>

#### 继承选择器
规则是，子标记在没有定义的情况下所有的样式都是继承父标记的，当子标记重复定义了父标记定义过的声明时，子标记就执行后面的声明；与父标记不冲突的地方仍然沿用父标记的声明。

	<!DOCTYPE HTML>
	<html>
	<head>
	<meta charset="utf-8">
	<title></title>
	<style>
		h1 {
			color: red;
			text-decoration: underline;
		}
		h1 strong {
			color: #004400;
			font-size: 40px;
		}
		</style>
	</head>
	<body>
	<h1>等待太久得来的<strong>东西</strong></h1>
	<h1>多半已经不是当初自己<font>想要</font>的样子了</h1>
	</body>
	</html>

{% iframe http://7xibui.com1.z0.glb.clouddn.com//uploads/web_dev/css_study/1_3_6.html %}

#### 伪类选择器
伪类作用在标记的状态上，包括:first-child, :link, :visited, :hover, :active, :focus, :lang等等。

伪类选择符定义的样式最常用在标记`<a>`标记上，它表示链接的4中不同状态：未访问链接(:link)、已访问链接(:visited)、激活状态(:active)和鼠标停留在链接上(:hover)。

	<!DOCTYPE HTML>
	<html>
	<head>
	<meta charset="utf-8">
	<title>伪类</title>
	<style>
		a:link { color: red }
		a:visited { color: green}
		a:hover { color: blue }
		a:active { color: orange }
		</style>
	</head>
	<body>
	<a href="http://www.netcan.xyz">访问NetCan_Space</a>
	</body>
	</html>

{% iframe http://7xibui.com1.z0.glb.clouddn.com//uploads/web_dev/css_study/1_3_7.html %}

#### 属性选择器
直接通过标记的属性来进行修饰，就是根据某个属性是否存在或属性值来寻找元素。

	<!DOCTYPE HTML>
	<html>
	<head>
	<meta charset="utf-8">
	<title></title>
	<style>
	[align="left"] {
	font-size: 20px;
	font-weight: bolder;
	}
	[lang^="zh"] {
	color:blue;
	font-size: 25px;
	}
	[href$="xyz"] {
	color: red;
	font-size: 30px;
	}
	</style>
	</head>
	<body>
	<p>沧海月明珠有泪</p>
	<p>蓝田日暖玉生烟</p>
	<p>此情可待成追忆？只是当时已惘然。</p>
	<a href="http://www.netcan.xyz">访问NetCan_Space</a>
	</body>
	</html>

{% iframe http://7xibui.com1.z0.glb.clouddn.com//uploads/web_dev/css_study/1_3_8.html %}

### 选择器声明

#### 集体声明
在一个页面中，有时候需要将不同种类的标记样式保持一致，例如需要p标记和h1字体保持一致，此时可以使p和h1标记共同使用类选择器，或者使用集体声明方法。

	<!DOCTYPE HTML>
	<html>
	<head>
	<meta charset="utf-8">
	<title></title>
	<style>
	h1,h2,h3,p {
		color: blue;
		font-size: 20px;
		font-weight: bolder;
	}
	</style>
	</head>
	<body>
	<h1>国破山河在，城春草木深。</h1>
	<h2>感时花溅泪，恨别鸟惊心。</h2>
	<h3>烽火连三月，家书抵万金。</h3>
	<p>白头搔更短，浑欲不胜簪。 </p>
	</body>
	</html>

{% iframe http://7xibui.com1.z0.glb.clouddn.com//uploads/web_dev/css_study/1_4_1.html %}

#### 多重嵌套声明
在通过css控制html样式时，还可以使用层层递进的方式，即嵌套方式，例如当`<p>`和`</p>`标记之间包含`<a></a>`标记时，就可以使用这种方式对html标记进行修饰。具体案例看[3.6](#继承选择器 "继承选择器")。

### css继承
具体请看[3.6](#继承选择器 "继承选择器")。
