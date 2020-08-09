title: 手机编程游戏Hacked通关攻略
toc: true
original: true
date: 2016-01-26 14:25:49
tags:
- 经验
categories:
- 随笔
---
{% raw %}
<style type="text/css">
.article img {
	width: 45%;
}
</style>
{% endraw %}

## 介绍
最近玩通了手游[Hacked](http://www.hackedapp.com)，现在安利给大家。

该游戏[Hacked](http://www.hackedapp.com/)，利用内置的H语言观察每关的输入输出样例推测算法并编写程序通过任务。游戏涉及素数回文串判定，进制转换，手排，栈，树，递归等算法。还能自定义编写游戏，绘图。。内置robot机器人可互动，例如走迷宫最短路问题。。游戏还能联网下载其他人的游戏，或者发布自己的游戏，可惜需要`Twitter`，正在注册，卡在手机验证一步。

下面是截图（来自官方）：
![Hacked](http://7xibui.com1.z0.glb.clouddn.com/%40%2F2016%2F01%2F2968096_orig.png) ![自定义游戏](http://7xibui.com1.z0.glb.clouddn.com/%40%2F2016%2F01%2F6702899_orig.png)

下面是吐槽：
* 自定义函数不支持外部全局变量
* 逻辑操作符`&&`当第一个表达式为假的时候还会计算第二个表达式。。作者`C`语言不过关啊。。
* 支持`Lambda`函数。。

## 攻略

### The Hackpad

#### Increment me
根据输入可知，输出为输入`+1`
```cpp
input + 1;
```

#### Positive
判断输入是否为非负。
```cpp
input > 0;
```

#### Absolute
求出输入的绝对值。
```cpp
if input < 0 {
	input = -input;
}
return input;
```

#### Absolute 2
和上一题一样，只是利用函数`abs`。
```cpp
abs(input);
```

### High School Hack

#### Power
求出输入的平方值。
```cpp
while var_a < input {
	var_b = var_b + input;
	var_a++;
}
return var_b;
```

#### Power 2
同上，只是利用函数。
```cpp
pow(input, 2);
```

#### Length
求出输入的表长。
```cpp
foreach var_a in input {
	var_b++;
}
return var_b;
```

#### Push it
根据输入，输出表`[0, 1, ..., input-1]`
```cpp
var_b = [];
while var_a < input {
	var_b.push(var_a);
	var_a++;
}
return var_b;
```

### Jailbreak

#### Max
求出输入的两个数的最大值。
```cpp
if input[0] > input[1] {
	return input[0];
}
else {
	return input[1];
}
```

#### Maxxxx
求出输入的表的最大值。
```cpp
foreach var_a in input {
	var_b = max(var_a, var_b);
}
```

#### This is odd
判断是不是奇数，是奇数输出1否则输出0。
```cpp
input - input / 2 * 2;
```

#### A man, a plan, ...
判断是不是回文串。
```cpp
while var_a < input.length {
	if input[var_a] != input[input.length - 1 - var_a] {
		return false;
	}
	var_a++;
}
return true;
```

### Cheatcode

#### Bring some order
将输入的字符串升序。
```cpp
while var_a < input.length {
	var_b = var_a + 1;
	while var_b < input.length {
		if input[var_a] > input[var_b] {
			var_c = input[var_a];
			input[var_a] = input[var_b];
			input[var_b] = var_c;
		}
		var_b++;
	}
	var_a++;
}
return input;
```

#### Missing numbers
将表中缺的数字找出来。
```cpp
input.sort;
var_c = [];
while var_a < input[input.length - 1] {
	if var_a < input[var_b] {
		var_c.push(var_a);
	}
	else {
		var_b++;
	}
	var_a++;
}
return var_c;
```

#### Anagrams
判断表中的每组字符集是否互相相等。
```cpp
while var_a < input.length - 1 {
	var_b = input[var_a].sort;
	var_c = input[var_a+1].sort;
	if var_b.length != var_c.length {
		return false;
	}
	while var_d < var_b.length {
		if var_b[var_d] != var_c[var_d] {
			return false;
		}
		var_d++;
	}
	var_a++;
}
return true;
```

### Corrupted

#### 11010111010100...
将二进制转换为十进制。
```cpp
foreach var_a in input {
	var_b = var_b*2 + var_a;
}
```

#### Prime
判断输入是否为素数。
```cpp
if input == 1 {
	return false;
}
var_a = 2;
while var_a * var_a < input + 1 {
	if mod(input, var_a) == 0 {
		return false;
	}
	var_a++;
}
return true;
```

#### Number in order
判断输入的字符串是否升序。
```cpp
while var_a < input.length - 1 {
	if input[var_a] > input[var_a+1] {
		return false;
	}
	var_a++;
}
return true;
```

### Cyber Attack

#### Complete
补齐输入表中缺的数字。
```cpp
var_b = [];
while var_a < input[input.length - 1] + 1 {
	var_b.push(var_a);
	var_a++;
}
return var_b;
```

#### Match
感谢[@堂堂卢鹏浩](https://www.zhihu.com/people/lu-peng-hao-95)的提醒，`())(`的情况我的代码就是错的了，现已更正如下：
```cpp
foreach var_a in input {
	if var_a == "(" {
		var_b++;
	}
	else {
		var_b--;
		if var_b < 0 {
			return false;
		}
	}
}
return var_b == 0;
```

~~判断输入的括号是否匹配。很奇怪为什么要判断第一个是否为`)`，按照我的算法不判断也能过啊。~~

```cpp
if input[0] == ")" {
	return false;
}
foreach var_a in input {
	if var_a == "(" {
		var_b++;
	}
	else {
		var_b--;
	}
}
return var_b == 0;
```

#### Rotate
将输入表的所有元素向左移动一位。
```cpp
var_b = input[0];
while var_a < input.length - 1 {
	input[var_a] = input[var_a + 1];
	var_a++;
}
input[input.length - 1] = var_b;
return input;
```

### Nuclear Plant

#### Add one
将表中所有元素值`+1`。
```cpp
while var_a < input.length {
	input[var_a]++;
	var_a++;
}
return input;
```

#### Positivity
将表中非负数化为`true`，负数为`false`。
```cpp
var_b = [];
foreach var_a in input {
	var_b.push(var_a > -1);
}
```

#### Nearest to [0, 0]
求出表中坐标谁更接近原点`[0, 0]`。自定义函数上场了。。
```cpp
function f1: var_a, var_b {
	if pow(var_a[0], 2) + pow(var_a[1], 2) > pow(var_b[0], 2) + pow(var_b[1], 2) {
		return var_b;
	}
	else {
		return var_a;
	}
}
while var_a < input.length - 1 {
	var_b = f1(input[var_a], input[var_a + 1]);
	var_a++;
}
return var_b;
```

### Killer Robot

#### Addition
将表中的两组数化为十进制形式并相加得到的结果一位一位地拆成表。例：

	[[1, 2], [1, 9]] => 12+19=31 => [3, 1]

```cpp
function f1: var_a {
	foreach var_b in var_a {
		var_c = var_c * 10 + var_b;
	}
	return var_c;
}
var_a = f1(input[0]) + f1(input[1]);
var_b = [];
while var_a > 0 {
	var_b.push(mod(var_a, 10));
	var_a = var_a / 10;
}
var_c = [];
while var_b.length > 0 {
	var_c.push(var_b.pop());
}
return var_c;
```

#### Match 2
也是括号匹配问题。。同样很奇怪为啥要判断第一个括号的值。
```cpp
var_b = [];
if input[0] == ")" || input[0] == "]" {
	return false;
}
foreach var_a in input {
	if var_a == "(" || var_a == "[" {
		var_b.push(var_a);
	}
	if var_a == ")" {
		if var_b.pop() != "(" {
			return false;
		}
	}
	if var_a == "]" {
		if var_b.pop() != "[" {
			return false;
		}
	}
}
return var_b.length == 0;
```

### Skynet

#### Three
将表中所有数字提取出来组成一张新表。递归应用。。
```cpp
var_b = [];
function f1: var_a, var_b {
	if var_a.is_list() {
		foreach var_c in var_a {
			f1(var_c, var_b);
		}
	}
	else {
		var_b.push(var_a);
	}
}
f1(input, var_b);
return var_b;
```

### Retirement

#### Draw
根据输入的坐标绘制点。
```cpp
draw(input[0], input[1]);
```
