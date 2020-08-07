title: PHP字符串处理
toc: false
original: true
tags:
  - PHP
categories:
  - 编程
date: 2015-03-10 16:56:53
updated: 2015-03-10 16:56:53
---
`.`用于连接两字符串

`strlen()`计算字符串长度，例如`strlen($string)`

`str_world_count()`计算字符串中的单词数，无法计算中文字符

`ltrim()`, `rtrim()`, `trim()`分别去掉字符串左边，右边，两边多余的空格

`explode()`和`implode()`分别为切分(返回数组)和组合(合并数组)字符串函数，

	str=”HelloWorld”,a = explode(‘_’, str),str=implode(‘_’, $a)

`substr(目标字符串, 起始位置, 截取长度)`在字符串中截取子串，若截取长度小于0，则截取除了从目标字符串尾部算起的长度数的字符以外的所有字符串

`substr_replace(目标字符串, 替换字符串, 起始位置, 替换长度)`字符串子串替换

`strstr(目标字符串, 需要查找的字符串)` 返回从第一个查到字符串的位置往后所有的字符串内容。`stristr()`对大小写不敏感

`ereg(正则表达式规范, 目标字符串, 数组)`在目标字符串中寻找符合正则表达式的字符串子串，找到了则存在一个数组中，`eregi()`对大小写不敏感。(此函数在php5.3上没用)

`ereg_replace(正则表达式, 欲取代的字符串子串, 目标字符串)`正则表达式替换字符串子串，`eregi_replace()`对大小写不敏感。(此函数在php5.3上没用)

`split(正则表达式, 目标字符串)`以正则表达式把目标字符串切分成若干子串存入数组。`spliti()`对大小写不敏感。(此函数在php5.3上没用)

`nl2br($string)`将$string中的\n生成
