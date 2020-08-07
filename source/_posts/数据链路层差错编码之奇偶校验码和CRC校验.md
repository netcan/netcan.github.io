title: 数据链路层差错编码之奇偶校验码和CRC校验
toc: true
original: true
date: 2016-03-11 19:59:32
tags:
- 计算机网络
categories:
- 学习笔记
---

刚刚到操场完成了大学第一个5公里慢跑，现在神清气爽。

今天学到了数据链路层，看到了各种差错编码，例如奇偶校验码，循环冗余校验码（`CRC`）等等，都是在数据后面加个冗余信息用于判断。

	D => DR(R为差错检验或者纠正比特)

## 奇偶校验码
奇偶校验码比较简单了，就是在一串数据后面加一位，用来凑够奇数个（奇校验）或偶数个（偶校验）`1`。

举个栗子：
### 奇校验:
	01100110 => 01100110(1)

### 偶校验
	01100110 => 01100110(0)

### 二维奇偶校验
二维奇偶校验就是同时检验每行，每列的奇偶校验，可以纠错。
<!--more-->

{% raw %}
$$
\begin{array}{ccc|c}
d_{1,1} & \cdots & d_{1,j} & d_{1,j+1} \\
d_{2,1} & \cdots & d_{2,j} & d_{2,j+1} \\
\cdots & \cdots & \cdots & \cdots \\
d_{i,1} & \cdots & d_{i,j} & d_{i,j+1} \\ \hline
d_{i+1,1} & \cdots & d_{i+1,j} & d_{i+1,j+1}
\end{array}
$$
{% endraw %}

举个栗子（下面都是偶校验）：
#### 无错情况
{% raw %}
$$
\begin{array}{ccccc|c}
1&0&1&0&1&1\\
1&1&1&1&0&0\\
0&1&1&1&0&1\\ \hline
0&0&1&0&1&0
\end{array}
$$
{% endraw %}

#### 有错&纠错
{% raw %}
$$
\begin{array}{ccccc|c}
1&0&1&0&1&1\\
1&\color{red}{0}&1&1&0&0\\
0&1&1&1&0&1\\ \hline
0&0&1&0&1&0
\end{array}
$$
{% endraw %}

## CRC校验
这才是重头戏，检错能力极强，据说准确率可达`99%`。先来看看它的算法：
1. 将原数据比特$D$视为一个二进制数（本来就是二进制）
2. 发送接收双方约定一个$r+1$位的生成函数$G$，也是用二进制表示。
3. 找出一个$r$位`CRC`比特$R$，附带在$D$后面用于冗余校验码。

检错的时候，就看看$G$能否整除$DR$，若余式全0，无错；否则有错。

需要注意的是这里的整除，并不是平常所用的除法，而是**模2除法**，就是长除的时候减法用**异或**来计算，也就是不进位。

所以可以这样来找$R$:

{% raw %}
$$
\begin{align}
& D\cdot 2^r \oplus R = n G \Rightarrow(两边异或)\\
& D\cdot 2^r = n G \oplus R \\
\end{align}
$$
{% endraw %}

因为这里是用**异或**来代替加减法（不进位），所以上式可看做加法，那么$R$就是通过$D\cdot 2^r$（$\cdot 2^r$相当于$D$左移$r$位）除以$G$得到的余式。

例如：
### 求R
$D=101110$，$G=x^3+1(1001)$，
{% raw %}
$$
\require{enclose}
\begin{array}{rl}
    101011 & \hbox{(模二除法)}\\[-3pt]
   1001 \enclose{longdiv}{101110\color{red}{000}}\kern-.2ex \\[-3pt]
      \underline{1001\phantom{00000}} \\[-3pt]
      1010\phantom{000} \\[-3pt]
      \underline{1001}\phantom{000}\\[-3pt]
      1100\phantom{0}  \\[-3pt]
      \underline{1001}\phantom{0}\\[-3pt]
      1010  \\[-3pt]
      \underline{1001}\\[-3pt]
      \phantom{00}\color{red}{011}
  \end{array}
$$
{% endraw %}

最后求得$R=011$。
### 校验
最后来校验一下所求的结果是否正确，就是拿$101110\color{red}{011}$除以$1001$看看余式是否为0。

{% raw %}
$$
\require{enclose}
\begin{array}{rl}
    101011\\[-3pt]
   1001 \enclose{longdiv}{101110\color{red}{011}}\kern-.2ex \\[-3pt]
      \underline{1001\phantom{00000}} \\[-3pt]
      1010\phantom{000} \\[-3pt]
      \underline{1001}\phantom{000}\\[-3pt]
      1101\phantom{0}  \\[-3pt]
      \underline{1001}\phantom{0}\\[-3pt]
      1001  \\[-3pt]
      \underline{1001}\\[-3pt]
      0
  \end{array}
$$
{% endraw %}

结果是显然的。。

