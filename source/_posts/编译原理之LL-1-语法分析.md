---
title: 编译原理实验之LL(1)语法分析器设计
toc: true
top: false
original: true
date: 2016-10-09 16:08:50
tags:
- 编译原理
- C/C++
categories:
- 学习笔记
---

编译原理第二个实验是LL(1)语法分析，这里我来写一下关于LL(1)的内容，虽然书上都有，但是总结一遍，应该印象更加深刻的，也有助于写代码。目前LL(1)实验已全部完成，项目地址：[http://115.159.147.250:666/LL1/](http://115.159.147.250:666/LL1/)，代码地址：[https://github.com/netcan/compilingTheory](https://github.com/netcan/compilingTheory)，效果图：
![https://raw.githubusercontent.com/netcan/compilingTheory/master/ll1.gif](https://raw.githubusercontent.com/netcan/compilingTheory/master/ll1.gif)


我写这篇博文写了好几天了，描述比较凌乱，建议大家还是看书吧，或者直接看我[程序设计部分](#程序设计)。一定要搞懂$first$集和$follow$集的求法，不然写程序也会遇到困难的，这里有篇不错的关于求$first$和$follow$集的论文，推荐看看：[https://www.cs.virginia.edu/~weimer/2008-415/reading/FirstFollowLL.pdf](https://www.cs.virginia.edu/~weimer/2008-415/reading/FirstFollowLL.pdf)

LL(1)文法主要是求$first$集和$follow$集，个人觉得$follow$集比较麻烦点，然后就是用这两个集合求预测分析表。

## $first$集
$first$集就是求文法符号串$\alpha$的开始符号集合$first(\alpha)$，例如有以下文法$G(E)$：
{% raw %}
$$ 
E\to TE' \\
E'\to +TE'|\varepsilon\\
T\to FT'\\
T'\to *FT'|\varepsilon\\
F\to (E)|\bf{id}
$$
{% endraw %}

用例子还是比较好说明的，很容易求出各非终结符的$first$集。
{% raw %}
$$
first(E) = first(T) = first(F) = \{(, \bf{id}\}\\
first(E') = \{+, \varepsilon\}\\
first(T') = \{*, \varepsilon\}
$$
{% endraw %}

这里给出$first$集的一般描述。

{% raw %}
$$
FIRST(\alpha) =\{t|(t\ is\ a\ terminal\ and\ \alpha\Rightarrow^∗ t\beta)or(t=\varepsilon\ and\ \alpha \Rightarrow^* \varepsilon)\}
$$
{% endraw %}

也就是说，如果非终结符$E$有产生式$T\beta$，那么它们$first(E)=first(T)$，这个不难理解，因为我（$E$）能推出你（$T$），你又能推出它（开始符号，终结符），那我们都能推出它。

$first$集的作用是，当用来匹配开头为$a$的文本时，有产生式$A\to X\alpha|Y\beta$，若$a\in X$，则选择候选式$A\to X\alpha$，若$a\in Y$，选择$A\to Y\beta$，说了那么多，我只想说，不是用$first(A)$这个来匹配$a$，而是用它的候选式$first(X)$或者$first(Y)$来匹配。

用上面的例子来说，匹配$(233$，因为$(\in first(TE')$，应该选择$E'\to +TE'$这个候选式。

## $follow$集
$follow$集比较抽象，它用来解决这个问题的，如果非终结符$E'$的$first(E')$含有$\varepsilon$，那么选择会不确定，比如说$G(E)$，
{% raw %}
$$
E\to Tb|F\\
T\to a|\varepsilon\\
F\to c
$$
{% endraw %}

很容易求得
{% raw %}
$$
first(Tb) = \{a, \varepsilon\} \\
first(F) = {c}
$$
{% endraw %}

匹配开头为$a$，因为$a\in first(Tb)$，选择$E\to Tb$产生式，匹配开头为$c$，因为$c\in first(F)$，选择$E\to F$产生式。然而当我匹配$b$时，因为$b\not \in first(Tb)\land \not \in first(F)$，这时候就不知道选择哪个产生式了，但是因为$\varepsilon\in first(Tb)$，且$E\to Tb|F\Rightarrow (a|\varepsilon)b|F\Rightarrow ab|b|F$，应该选择$E\to Tb$的，这说明了$first$集的不足，从而引进$follow$集。

给出定义，$follow(A)$即为非终结符$A$后面可能接的符号集。

至于怎么求$follow(A)$，我就直接摘抄PPT的定义吧，然后在说明。

连续使用下面的规则，直至每个$follow$不再增大为止。
* 对于文法的开始符号$S$，置#于$follow(S)$中。
* 若$A\to \alpha B\beta$是一个产生式，则把$first(\beta)\backslash \{\varepsilon \}$加至$follow(B)$中；
* 若$A\to \alpha B$是一个产生式，或$A\to \alpha B\beta $是一个产生式而$\varepsilon\in FIRST(\beta)$，则把$follow(A)$加至$follow(B)$中。 

第三条规则可以这么理解，因为$B$后面啥都没，$A$又能推出$B$，所以应该把$A$后面的符号（$follow(A)$）加入$follow(B)$中。需要注意的是，$follow$集不含$\varepsilon$。

所以要求某个非终结符的$follow$集，只要在**产生式右边**中找，然后根据它后面一个**符号**按照上述规则计算就行了。

## 预测分析表
求得$first$集和$follow$集后，求分析表就比较容易了。这里简单说下构建方法。

对文法的每个产生式，执行下面步骤。

* 对$first(\alpha)$的每个**终结符**$a$，将候选式$A\to \alpha$加入$M[A, a]$
* 如果$\varepsilon\in first(\alpha)$，把$follow(A)$的每个**终结符**$b$（包括#），把$A\to \alpha$加入$M[A, b]$。

## 程序设计
代码的核心部分就是求$first$集和$follow$集了，这也是程序的精髓所在。

首先我约定字符@代表$\varepsilon$，因为键盘上没那个符号，所以随便找了个合适的符号代替。约定单个大写字母代表非终结符，小写字母或某些符号代表终结符。

然后设计一个叫做产生式的类`Prod`，它的构造函数接受一个字符串，字符串描述了一个产生式。然后分割字符串，把它的非终结符存入`noTerminal`成员中，候选式存入`selection`集合中。

接着设计一个叫`LL1`的类，也就是核心部分了，它提供了求`first`、`follow`、分析表等方法。因为`first`集和`follow`集可能会求未知表达式的`first`集和`follow`集，比如说`A->B，B->C`，欲求`first(A)`，需求`first(B)`，欲求`first(B)`，需求`first(C)`，从而确定了这两种集合求法只能用递归来求，这也是我所能想到的最简单求法，可以参考我代码。

然而现在（2016/10/21）补充下，经过反复调教，`first`集都重写了5次，而`follow`集是**不能用递归**来求的，因为有可能出现这种情况：
{% raw %}
$$
A\to bAS\\
S\to aA\\
S\to d\\
A\to \varepsilon
$$
{% endraw %}

求`follow(A)`需要`first(S)`和`follow(S)`，递归求`follow(S)`需要`follow(A)`，然而这样就进入了死递归。所以最后我改写成5个循环搞定。


求出了`first`和`follow`，剩下的就好办了。至于图形界面，和[上次](/2016/10/07/编译原理之词法分析器设计/)一样，套了个`php`，通过`php`传json数据到前端，前端输入数据取数据，渲染页面。那颗语法树的画法，通过前序遍历画得。

