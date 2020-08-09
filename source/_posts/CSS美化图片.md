title: CSS美化图片
toc: true
original: true
tags:
  - CSS
categories:
  - 学习笔记
date: 2015-04-05 16:49:07
updated: 2015-04-05 16:49:07
---

好久没写博客了，最近在忙着编写百科知识竞赛平台，第一个php项目实现了。。有点不爽的是，希望这个百科竞赛是校级的，不然废了那么多精力最后只是系级的比赛很不值，但也算作是经验吧。。只是写的很凌乱。。

把HTML+CSS的书看完了，来继续整理下css的知识。

### 修饰图片

#### 图片边框([border-style](http://www.w3school.com.cn/cssref/pr_border-style.asp))
可定义边框样式，即边框风格。

	{ border-style: dotted|dashed|solid|double; }
	{ border-top-style|border-right-style|border-bottom-style|border-left-style: dotted|dashed|solid|double; }

#### 图片缩放

使用([max-width](http://www.w3school.com.cn/cssref/pr_dim_max-width.asp))和([max-height](http://www.w3school.com.cn/cssref/pr_dim_min-height.asp))，分别用来设置图片的宽度最大值和高度最大值。例如

	{ max-height: 180px; }

使用([width](http://www.w3school.com.cn/cssref/pr_dim_width.asp))和([height](http://www.w3school.com.cn/cssref/pr_dim_height.asp))属性，可以设置图片的宽度和高度。

### 对齐图片

#### 横向对齐方式

如果要定义图片的对齐方式，不能在样式表中直接定义图片样式，需要在图片的上一个标记级别，即父标记定义，让图片继承父标记的对齐方式。毕竟图片本身没有对齐属性，需要使用css继承父标记的([text-align](http://www.w3school.com.cn/cssref/pr_text_text-align.asp))来定义对齐方式。

#### 纵向对齐方式

使用(vertical-align)属性来定义图片纵向的对齐方式。

	{vertical-align: baseline|sub|super|top|text-top|middle|bottom|text-bottom|length|%;}

### 图文混排

#### 文字环绕

使用([float](http://www.w3school.com.cn/cssref/pr_class_float.asp))属性来定义该效果，该属性主要定义元素在哪个方向浮动，一般情况下该属性总应用于图像，使文本环绕在图像四周，有时候也可以定义其他浮动元素。

	{float: none|left|right; }

#### 设置图片与文字间距

可以使用([padding](http://www.w3school.com.cn/cssref/pr_padding.asp))来设置。

padding属性主要用来在一个声明中设置所有内边距属性，既可以设置元素所有内边距的宽度，或者设置各边上的内边距的宽度。

	{ padding: padding-top|padding-right|padding-bottom|padding-left; }
