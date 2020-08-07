title: PHP与Web页面交互
toc: false
original: true
tags:
  - PHP
categories:
  - 编程
date: 2015-03-15 23:32:57
updated: 2015-03-15 23:32:57
---

PHP基本上搞定了，还差最后一章PHP图形处理。

这节也通过一堆代码来写吧，很好理解的。

首先是web页面，file.html页面用来测试收集数据和上传文件。
<!--more-->
```html
<html>
<head>
	<meta charset="utf-8">
	<title>测试PHP与Web交互与文件/目录操作</title>
</head>
<body>
<center>
	<h2>测试PHP与Web交互与文件/目录操作</h2>
	<form action="file.php" method="post">
		<table>
			<tr bgcolor="#3399ff">
				<td>用户名:</td><td><input type="text" name="name" size="20" maxLength="20" /></td>
			</tr>
			<tr bgcolor="#CCCCCC">
				<td>性别:</td><td><center><input type="radio" name="sex" value="1" checked/>男<input type="radio" name="sex" value="2" />女</center></td>
			</tr>
			<tr bgcolor="#4ADDFF">
				<td>擅长:</td>
				<td>
				<center>
					<select name="fav" >
						<option value="1">C/C++</option>
						<option value="2">Java</option>
						<option value="3">PHP</option>
						<option value="4">Python</option>
						<option value="5">HTML5/CSS</option>
					</select>
				</center>
				</td>
			</tr>
			<tr bgcolor="#CAFF0D">
				<td>密码:</td><td><input type="password" name="passwd" size="20" maxLength="20"/></td> 
			</tr>
			<tr bgcolor="#CAFF0D">
				<td>确认:</td><td><input type="password" name="repasswd" size="20" maxLength="20"/></td> 
			</tr>
			<tr bgcolor="#E8950C">
				<td colspan=2><center><input type="reset" value="重置"/>
						<input type="submit" value="提交"/></center></td>
			</tr>
		</table>
	</form>
	<h2>测试文件上传</h2>
	<form method="post" action="upload.php" enctype="multipart/form-data">
		<input type="file" name="upload"/>
		<input type="submit" value="上传"/>
	</form>


</body>
</html>
```
其次是file.php，处理表单接收的数据，并写入文件file.data，然后读取。

```php
<meta charset="utf-8">
<?php
   echo "<h2>测试PHP与Web交互与文件/目录操作</h2>";
   $name = trim($_POST['name']);
   $sex = trim($_POST['sex']);
   $fav = trim($_POST['fav']);
   $fav = trim($_POST['fav']);
   $passwd = trim($_POST['passwd']);
   $repasswd = trim($_POST['repasswd']);
   if($passwd != $repasswd) {
      echo "<script>alert(\"密码不一致!\");history.go(-1);</script>";
	  exit();
   }
   @$sav = fopen("./file.data", "a+b");
   fwrite($sav, date("Y-M-D H:i:s")."\t$name\t$sex\t$fav\t$passwd\n");
   rewind($sav);
   echo "信息已提交!<br>";
   while(!feof($sav))
      echo fgets($sav)."<br>";
   fclose($sav);
?>
```
最后是上传的upload.php，用来处理上传的文件，和操作目录。

```php
<meta charset="utf-8">
<?php
   define("UPLOAD", "./uploads/");
   echo "<h2>测试文件上传</h2>";
   echo "上传文件名: ".$_FILES['upload']['name']."<br>";
   echo "上传文件类型: ".$_FILES['upload']['type']."<br>";
   echo "上传文件大小: ".$_FILES['upload']['size']/1024;
   echo "kb<br>";
   if(!file_exists(UPLOAD))
	  mkdir(UPLOAD);
  move_uploaded_file($_FILES['upload']['tmp_name'], iconv("UTF-8", "GB2312",UPLOAD.$_FILES['upload']['name']));
  echo "当前目录:".getcwd()."<br>"; 
  echo "目录文件列表："."<br>";
  print_r(scandir(UPLOAD));
  echo "<br>===========================<br>";
  $d = dir(UPLOAD);
  while($f = $d->read())
  echo iconv("GB2312", "UTF-8",$f)."<br>";
  $d->close();
  echo "<br>===========================<br>";
  $d = opendir(UPLOAD);
  while(false !== ($f = readdir($d)))
     echo iconv("GB2312", "UTF-8",$f)."<br>";
   closedir($d);
?>
```
最终效果图：
![http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob2.png](http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob2.png)

文件上传：
![http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob5.png](http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob5.png)
![http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob6.png](http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob6.png)



