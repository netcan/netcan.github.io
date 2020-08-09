title: CSS字体与段落属性
toc: true
original: true
tags:
  - CSS
categories:
  - 学习笔记
date: 2015-03-27 22:43:41
updated: 2015-03-27 22:43:41
---

点击属性可跳转到对应的w3school手册。

### css文字样式

#### 字体([font-family](http://www.w3school.com.cn/cssref/pr_font_font-family.asp))
font-family属性用于指定字体的类型，如宋体、黑体、Times New Roman等，既在网页中展示字体的不同形状，语法如下。

	{font-family: name;}
	{font-family: cursive|fantasy|monospace|serif|sans-serif;}

#### 字号([font-size](http://www.w3school.com.cn/cssref/pr_font_font-size.asp))
通常使用font-size设置文字大小。

	{font-size: 数值|inherit|xx-small|x-small|small|medium|large|x-large|xx-large|larger|smaller|length;}

#### 字体风格([font-style](http://www.w3school.com.cn/cssref/pr_font_font-style.asp))
定义字体的风格，即字体的显示样式。

	{font-style: normal|italic|oblique|inherit;}

#### 字体加粗([font-weight](http://www.w3school.com.cn/cssref/pr_font_weight.asp))
定义字体的粗细程度。

	{font-weight: 100-900|bold|bolder|lighter|normal;}

#### 小写字母转为大写字母([font-variant](http://www.w3school.com.cn/cssref/pr_font_font-variant.asp))
属性设置小型大写字母的字体显示文本，这意味着所有的小写字母均会被转换为大写，但是所有使用小型大写字体的字母与其余文本相比，其字体尺寸更小。

	{font-variant: normal|small-caps|inherit;}

#### 字体复合属性([font](http://www.w3school.com.cn/cssref/pr_font_font.asp))
可以一次性使用多个属性的属性值定义文本字体。

	{font: font-style font-variant font-weight font-size font-family;}

其中font-size和font-family属性必须按照固定顺序出现，缺一不可。font-family有多个属性值的话用逗号隔开，前三个属性可以自由调换。

#### 字体颜色([color](http://www.w3school.com.cn/cssref/pr_text_color.asp))
设置字体的颜色。

	{color: color_name|hex_number|rgb(r,g,b)|inherit;}

### css段落样式

#### 单词间隔([word-spacing](http://www.w3school.com.cn/cssref/pr_text_word-spacing.asp))
设定词与词的间隔。

	{word-spacing: nomal|length;}

不能设定于文字间的间隔。

#### 字符间隔([letter-spacing](http://www.w3school.com.cn/cssref/pr_text_letter-spacing.asp))
设定字符间的距离。

	{letter-spacing: normal|length;}

#### 文本修饰([text-decoration](http://www.w3school.com.cn/cssref/pr_text_text-decoration.asp))
可以增加文字的下划线、顶划线、删除线等效果。

	{text-decoration: none|underline|blink|overline|line-through;}

#### 垂直对齐方式([vertical-align](http://www.w3school.com.cn/cssref/pr_pos_vertical-align.asp))
设定垂直的对齐方式。

	{vertical-align: baseline|sub|super|top|text-top|middle|bottom|text-bottom|%;}

#### 水平对齐方式([text-align](http://www.w3school.com.cn/cssref/pr_text_text-align.asp))
设定水平方向上的对齐方式。

	{text-align: left|right|center|justify|match-parent|inherit;}

#### 文本转换([text-transform](http://www.w3school.com.cn/cssref/pr_text_text-transform.asp))
可以将大写转换为小写，或者将小写转换成大写。

	{text-transform: none|capitalize|uppercase|lowercase;}

#### 文本缩进([text-indent](http://www.w3school.com.cn/cssref/pr_text_text-indent.asp))
可以使首行以给定的长度或百分比缩进。

	{text-indent: length;}

#### 文本行高([line-height](http://www.w3school.com.cn/cssref/pr_dim_line-height.asp))
设置行间距。

	{line-height: normal|length;}

#### 处理空白([white-spacing](http://www.w3school.com.cn/cssref/pr_text_white-space.asp))
设置对象内的空白字符(空格,TAB,换行)的处理方式。

	{white-spacing: normal|pre|nowrap|pre-wrap|pre-line|inherit;}

#### 文本反排([unicode-bidi](http://www.w3school.com.cn/cssref/pr_unicode-bidi.asp)和[direction](http://www.w3school.com.cn/cssref/pr_text_direction.asp))
可以更改文本的显示方向方向。direction设定文本流方向。

	{unicode-bidi: normal|bidi-override|embed;}
	{direction: ltr|rtl|inherit;}
