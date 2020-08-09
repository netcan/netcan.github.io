title: PHP操作MySQL数据库
toc: true
original: true
tags:
  - PHP
  - MySQL
  - 数据库
categories:
  - 编程
date: 2015-03-21 22:34:23
updated: 2015-03-21 22:34:23
---

### 使用`mysqli_connect()`函数连接到MySQL数据库

    mysqli_connect('MySQL服务器地址','用户名','密码','要连接的数据库');
Example:

	$db=mysqli_connect('localhost','root','password','test');

### 使用`mysqli_select_db()`函数选择数据库文件

    mysqli_select_db(数据库服务器连接对象,目标数据库名);

Example:

	$db=mysqli_connect('localhost','root','password');
	mysqli_select_db($db,'test');

### 使用`mysqli_query()`函数执行SQL语句

    mysqli_query(数据库服务器连接对象,SQL语句);
Example:

	$fruits=mysqli_query($db,"SELECT * FROM fruits");

#### 使用`mysqli_fetch_assoc()`函数从查询结果集中获取一行信息最为关联数组

    mysqli_fetch_assoc(SQL请求返回的对象)，将返回一个关联数组。
Example:

	$fruit = mysqli_fetch_assoc($fruits);
	echo $fruit['f_name'];

#### 使用`mysqli_fetch_object()`函数从查询结果集中获取一行信息作为对象

    mysqli_fetch_object(SQL请求返回的对象)，将返回一个对象。
Example:

	$fruit = mysqli_fetch_assoc($fruits);
	echo $fruit->f_name;

### 使用`mysqli_num_rows()`函数获取查询结果集中的记录数

    mysqli_num_rows(SQL请求返回的对象)，返回记录数目。
Example:

	fruit_nums = mysqli_num_rows($fruits);

### 使用`mysqli_free_result()`函数释放资源，

    mysqli_free_result(SQL请求返回的对象),释放查询结果资源。
Example:

	mysqli_free_result($fruits);

### 使用`mysqli_close()`函数关闭连接

    mysqli_close(需要关闭的数据库连接对象);
Example:

	mysqli_close($db);

### 简单的注册，查询实例
先上register_mysql.html页面文件，
```html
<html>
<head>
	<meta charset="utf-8">
	<title>PHP操作MySQL数据库</title>
</head>
<body>
<center>
	<h2>PHP操作MySQL数据库</h2>
	<form action="register_mysql.php" method="post">
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
			<tr bgcolor="#9966cc">
				<td>备注</td>
				<td><textarea name="info" cols="22" rows="3">在这里留下备注信息</textarea></td>
			</tr>

			<tr bgcolor="#E8950C">
				<td colspan=2><center><input type="reset" value="重置"/>
						<input type="submit" value="提交"/></center></td>
			</tr>
		</table>
	</form>
	<h2>MySQL数据库查询</h2>
	<form action="register_query.php" method="post">
		<table>
			<tr bgcolor="#FF0DFF">
				<td>用户名</td>
				<td><input type="text" size="20" maxLength="20" name="query_name"></td>
			</tr>
			<tr bgcolor="#4DFFC4">
				<td colspan=2><center><input type="reset" value="重置"/>
						<input type="submit" value="查询"/></center></td>
			</td>
		</tr>
	</table>
</form>
</body>
</html>
```
页面图：
![http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob10.png](http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob10.png)


其次是输入数据至数据库的register_mysql.php文件，

```php
<meta charset="utf-8">
<?php
   echo "<h2>PHP操作MySQL数据库</h2>";

   $name = trim($_POST['name']);
   $sex = trim($_POST['sex']);
   $fav = trim($_POST['fav']);
   $passwd = trim($_POST['passwd']);
   $repasswd = trim($_POST['repasswd']);
   $info = trim($_POST['info']);
   $sexa = array(0,"男","女");
   $fava = array(0,"C/C++语言","Java语言","PHP语言","Python语言","HTML5/CSS");
   $sex = $sexa[$sex];
   $fav = $fava[$fav];

   if($passwd != $repasswd) {
      echo "<script>alert(\"密码不一致!\");history.go(-1);</script>";
	  exit();
   }
   if(!$name || !$sex || !$fav || !$passwd || !$repasswd || !$info) {
	  echo "<script>alert(\"请填满数据!\");history.go(-1);</script>";
	  exit();
   }
   $user = mysqli_connect('localhost','root','');
   mysqli_select_db($user,'test');
   if(mysqli_connect_errno()) {
	  echo "<h3>MySQL数据库连接失败！</h3>";
	  exit();
   }
   else
   echo "<h3>MySQL数据库连接成功！</h3>";


   $insert = "INSERT INTO user(name,sex,fav,passwd,info) value('$name','$sex','$fav','$passwd','$info')";
   $insert = iconv("utf-8","gb2312",$insert);
   if(mysqli_query($user,$insert))
   echo "<h3>数据添加成功！</h3>";
   else
   echo "<h3>数据添加失败！</h3>";
   mysqli_close($user);
   echo "<input type=button onClick=\"javascript:history.go(-1)\" value=\"返回上一页\">";



?>
```
最后是查询数据的register_query.php。
```php
<meta charset="utf-8">
<?php
   $user = mysqli_connect('localhost','root','');
   mysqli_select_db($user,'test');
   if(mysqli_connect_errno()) {
	  echo "<center><h3>MySQL数据库连接失败！</h3></center>";
	  exit();
   }
   else
   echo "<center><h3>MySQL数据库连接成功！</h3></center>";
   mysqli_query($user,"set names utf8");
   $query_name = trim($_POST['query_name']);
   $query_users = mysqli_query($user,"SELECT * FROM user WHERE name LIKE '%$query_name%'");
   $query_users_num = mysqli_num_rows($query_users);
   echo "<center><table border=1>
		 <tr bgcolor=#4dffc4>
		 <td><center>ID</center></td><td><center>昵称</center></td><td><center>性别</center></td><td><center>擅长</center></td><td><center>密码</center></td><td><center>备注</center></td>
	  </tr>
	  ";
   for($i=0; $i<$query_users_num; ++$i) {
	  $query_user = mysqli_fetch_assoc($query_users);
	  echo "<tr bgcolor=4addff>
		 <td><center>$query_user[id]</center></td><td><center>$query_user[name]</center></td><td><center>$query_user[sex]</center></td><td><center>$query_user[fav]</center></td><td><center>$query_user[passwd]</center></td><td><center>$query_user[info]</center></td>
	  </tr>
		 ";
   }
   echo "</table>记录条数：$query_users_num
<input type=button onClick=\"javascript:history.go(-1)\" value=\"返回上一页\">";

?>
```
最终效果图，

提交数据：
![http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob11.png](http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob11.png)
查询数据：
![http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob12.png](http://7xibui.com1.z0.glb.clouddn.com/2015/03/blob12.png)
