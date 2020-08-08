title: 简易JavaScript计算器
toc: true
original: true
tags:
  - Javascript
categories:
  - 编程
date: 2015-03-10 12:03:54
---
## 乘法运算 
<script language="JavaScript">function mult() {document.multi.r.value = document.multi.a.value*document.multi.b.value;}</script>
<form name="multi"><input type="text" name="a" size="5" maxLength="5" value="1"/> * <input type="text" name="b" size="5" maxLength="5" value="1"/>= <input type="text" name="r" size="10" value="1" readonly/> <input type="button" value="计算" onClick="mult()"/> </form>

## 加法运算 
<script language="JavaScript">function ad() {document.add.r.value = parseFloat(document.add.a.value) + parseFloat(document.add.b.value);}</script>
<form name="add"><input type="text" name="a" size="5" maxLength="5" value="1"/> + <input type="text" name="b" size="5" maxLength="5" value="1"/>= <input type="text" name="r" size="10" value="1" readonly/> <input type="button" value="计算" onClick="ad()"/> </form>

<!-- more --> 

## 网页源代码(乘法)

```
<html>
<head>
    <meta charset="utf-8">
    <title>JavaScript乘法</title>
    <script language="JavaScript">
        function f() {
           document.x.r.value = document.x.a.value*document.x.b.value;
        }
     </script>
</head>
<body>
    <center><p>
        <h1>乘法运算</h1></p>
    <form name="x">
        <p><input type="text" name="a" size="5" maxLength="5" value="1"/> *
        <input type="text" name="b" size="5" maxLength="5" value="1"/>
        = <input type="text" name="r" size="10" value="1" readonly/> </p>
        <p><input type="button" value="计算" onClick="f()"/> </p>
    </form></center>
</body>
</html>
```
