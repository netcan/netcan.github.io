title: PHP类与异常
toc: false
original: true
tags:
  - PHP
categories:
  - 编程
date: 2015-03-15 00:40:08
updated: 2015-03-15 00:40:08
---

今晚总算把类和对象总结了出来，通过一个例子很好理解，我写了注释。
<!--more-->
```php
<?php
   class student { // 定义一个类
	  private $name;  // 私有属性
	  private $id;

	  function __construct($name="user", $id=123456) { // 构造函数,添加默认值
		 $this->name = $name;
		 $this->id = $id;
	  }
	  function __destruct() { // 构造函数
		 echo "该类".__CLASS__."被释放！<br>";
	  }
	  function get_name() { // 定义方法
		 return $this->name;
	  }
	  function get_id() {
		 return $this->id;
	  }
	  function set_name($name) {
		 $this->name = $name;
	  }
	  function set_id($id) {
		 $this->id = $id;
	  }
	  function __set($k, $v) { // 访问函数，可以通过其访问私有属性
		 $this->$k = $v; // 注意$this->$符号
	  }
	  function __get($v) {
		 return $this->$v;
	  }
   }
   final class Netcan extends student implements attr { // 继承类和接口类
	  private $height;
	  private $weight;
	  function __construct() {
		 $this->name = "Netcan";
		 $this->id = 666666;
		 $this->height = 172;
		 $this->weight = 120;
	  }
	  function __destruct() {
		 echo "该类".__CLASS__."被释放！<br>";
	  }
	  function show() { // 定义子类方法
		 echo "name: ".$this->name."<br>";
		 echo "id: ".$this->id."<br>";
		 echo "height: ".$this->height."<br>";
		 echo "weight: ".$this->weight."<br>";
	  }
   }
   interface attr { // 定义接口类
	  function show();
   }
   ###############测试类##################
   $s = new student("Netcan", 666666); // 注意创建对象的方法，$xx = new class();
   echo "name: ".$s->get_name()."<br>";
   echo "id: ".$s->get_id()."<br>";
   $s->set_name("hello"); // 调用方法修改私有属性
   $s->set_id(666666);
   echo "name: ".$s->get_name()."<br>";
   echo "id: ".$s->get_id()."<br>";
   $s->name = "world"; // 通过访问函数修改/访问私有属性
   $s->id = 123456;
   echo "name: ".$s->name."<br>";
   echo "id: ".$s->id."<br>";
   unset($s); // $s = null; // 销毁对象
   ##############测试子类###################
   $netcan = new Netcan(); // 创建子类对象
   $netcan->show(); // 调用子类方法
   unset($netcan);
   ##############测试异常##################
   # try {
   #	  throw
   #   } catch() {
   #
   # }
   ########################################
   try { 
	  if(ture)
	  	  throw new Exception("抛出异常!"); 
	   } catch(Exception $error) {    // 注意Exception后的$号
	   echo $error->getMessage()."<br>";
	   echo "File: ".$error->getFile()."<br>";
	   echo "Line: ".$error->getLine()."<br>";
	}
?>
```
输出：

	name: Netcan
	id: 666666
	name: hello
	id: 666666
	name: world
	id: 123456
	该类student被释放！
	name: Netcan
	id: 666666
	height: 172
	weight: 120
	该类Netcan被释放！

	Notice: Use of undefined constant ture - assumed 'ture' in F:\workspace\Netcan_web\study\class.php on line 80
	抛出异常!
	File: F:\workspace\Netcan_web\study\class.php
	Line: 81

好吧，我该去睡觉了= =
