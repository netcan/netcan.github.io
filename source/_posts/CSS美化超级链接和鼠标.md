title: CSS美化超级链接和鼠标
toc: true
original: true
tags:
  - CSS
categories:
  - 学习笔记
date: 2015-04-08 00:00:47
updated: 2015-04-08 00:00:47
---

### 超级链接特效

#### 改变超级链接的基本样式

对于超级链接的修饰，可以采用css伪类。

伪类|用途
-|-
a:link|定义a对象在未被访问前的样式
a:hover|定义a对象在鼠标悬停时的样式
a:active|定义a对象在被激活时的样式（在鼠标单击与释放之间发生的事件）
a:visited|定义a对象在其链接被访问过时的样式

例如

	a:hover { color: #545454; }

#### 超级链接背景图设置

通常使用([background-image](http://www.w3school.com.cn/cssref/pr_background-image.asp))属性来完成。例如

	a { background-image: url(image.png); }

### 鼠标特效

控制鼠标箭头，css通过改变cursor属性来实现对鼠标样式的控制。

属性值|说明
-|-
auto|自动，按照默认状态自行改变
crosshair|精确定位十字
default|默认鼠标指针
hand|手型
move|移动
help|帮助
wait|等待
text|文本
n-resize|箭头朝上双向
s-resize|箭头朝下双向
w-resize|箭头朝左双向
e-resize|箭头朝右双向
ne-resize|箭头朝右上双向
se-resize|箭头朝右下双向
nw-resize|箭头朝左上双向
sw-resize|箭头朝左下双向
pointer|指针
url|自定义鼠标指针

例如

	a { cursor: hand; }
