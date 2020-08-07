title: CSS设置背景与边框
toc: true
original: true
tags:
  - CSS
categories:
  - 学习笔记
date: 2015-04-06 15:35:16
updated: 2015-04-06 15:35:16
---

清明节都呆在宿舍了，博客也没怎么写，书都看完了= =

赶紧更新和室友打红警。。。

### 设置背景：

#### 背景颜色([background-color](http://www.w3school.com.cn/cssref/pr_background.asp))
默认背景色为透明(transparent)

	{ background-color： transparent|color; }
#### 背景图片([background-image](http://www.w3school.com.cn/cssref/pr_background-image.asp))
用于设定标记的背景图片。

	{ background-image: none|url(url); }
##### 背景图片重复([background-repeat](http://www.w3school.com.cn/cssref/pr_background-repeat.asp))
可设置图片的重复方式，包括水平重复，垂直重复和不重复等。
<!--more-->

	{ background-repeat: repeat|repeat-x|repeat-y|no-repeat; }
##### 背景图片显示([background-attachment](http://www.w3school.com.cn/cssref/pr_background-attachment.asp))
该属性用来设定背景图片是否随文档一起滚动。

	{ background-attachment: scroll|fixed; }
##### 背景图片的位置([background-position](http://www.w3school.com.cn/cssref/pr_background-position.asp))
指定背景图片在页面所处的位置。

	{ background-position: length|percentage|top|center|bottom|left|right; }
##### 设置图片大小([background-size](http://www.w3school.com.cn/cssref/pr_background-size.asp))

	{ background-size: [<length>|<percentage>|auto]|cover|contain; }
##### 背景复合属性([background](http://www.w3school.com.cn/cssref/pr_background.asp))
综合了以上所有与背景有关的属性，可以一次性地设定背景样式。

	{ background: [background-color][background-image][background-repeat][background-attachment][background-position][background-size][background-clip][background-origin]; }
### 设置边框

### 边框样式([border-style](http://www.w3school.com.cn/cssref/pr_border-style.asp))
主要用于为页面元素添加边框。

	{ border-style: none|hidden|dotted|dashed|solid|double|groove|ridge|inset|outset; }
#### 边框颜色([border-color](http://www.w3school.com.cn/cssref/pr_border-color.asp))
可以设置边框的颜色

	{ border-color: color; }
#### 边框线宽([border-width](http://www.w3school.com.cn/cssref/pr_border-width.asp))
设置边框的宽度。

	{ border-width: medium|thin|thick|length; }
#### 边框复合属性([border](http://www.w3school.com.cn/cssref/pr_border.asp))
集合了以上介绍的几种属性。

	{ border: border-width|border-style|border-color; }
