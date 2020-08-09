---
title: 编译原理实验之LR(1)分析器设计
toc: true
top: false
original: true
date: 2016-10-21 21:15:07
tags:
- 编译原理
- C/C++
categories:
- 学习笔记
---

终于做完第三个实验了，代码地址：[https://github.com/netcan/compilingTheory](https://github.com/netcan/compilingTheory)，项目地址：[http://115.159.147.250:666/LR/](http://115.159.147.250:666/LR/)，**第一次进入需要加载库，比较耗时，请耐心等待**，效果图：
![https://raw.githubusercontent.com/netcan/compilingTheory/master/LR.gif](https://raw.githubusercontent.com/netcan/compilingTheory/master/LR.gif)

先来吐槽下，据说这个实验是最难的一个实验，当然也是最后的一个实验了。在我们实验室上一届学长只有一个人写出来，可见其难度。

![http://7xqytu.com1.z0.glb.clouddn.com//2016/10/QQ%E6%88%AA%E5%9B%BE20161021212712.png](http://7xqytu.com1.z0.glb.clouddn.com//2016/10/QQ%E6%88%AA%E5%9B%BE20161021212712.png)

然而我并不觉得有什么难的，当我上课听懂老师讲LR语法分析的时候，我就疑惑了，难点在哪？学长说难在自动机的构建上，自动机比较难拍，不好用数据结构来描述。这当然给了我巨大信心，回去不到一天把LR分析器核心拍出来了，在没有参考任何代码、龙书，仅看着教科书上的算法来写的。

## 产生式`Prod`类
我来说下具体是怎么实现的吧，用面向对象来写比较好写，绝对比面向过程好写。先来看看我设计的最小的类`Prod`（为减小篇幅已删除无关紧要的函数）。

```cpp
class Prod { // 这里是存放形如X->abc的形式，不存在多个候选式
	private:
		char noTerminal; // 产生式左部非终结符名字
		string right; // 产生式右部
		set<char> additionalVt; // 附加终结符
		friend bool operator == (const Prod &a, const Prod &b) {
			return a.noTerminal == b.noTerminal && a.right == b.right;
		}
		friend bool operator == (const Prod &a, char c) {
			return a.noTerminal == c;
		}
	public:
		Prod(const char &noTerminal, const string& right, const set<char>& additionalVt):
			noTerminal(noTerminal), right(right), additionalVt(additionalVt) {}
};
```

这个类是存放产生式的，存放形如`A->Bc`，这里的`noTerminal`就是左边的`A`，`right`就是右边的`Bc`，而`additionalVt`是`LR(1)`的项目集的**搜索符**，长度为1所以叫`LR(1)`。我重载了`==`符号，后面用来搜索项目集/文法中是否有这个产生式简直不能再方便，构造函数也能快速构造产生式，无疑为后面项目集中插入产生式提供了方便。

## 项目集`Item`类
```cpp
class Item { // 项目集
	private:
		vector<Prod> prods; // 项目集产生式
		static set<char> Vn; // 非终结符
		static set<char> Vt; // 终结符
		static set<char> Symbol; // 所有符号
		friend bool operator == (const Item &a, const Item &b) {
			if(a.prods.size() != b.prods.size()) return false;
			else {
				for(const auto& p: a.prods) {// 选择a的每个产生式
					auto it = find(b.prods.begin(), b.prods.end(), p);
					if(it == b.prods.end())  // 找不到
						return false;
					else  {// 找到的话判断附加终结符是否都相等
						if(p.additionalVt != it->additionalVt)
							return false;
					}
				}
				return true;
			}
		}
	public:
		void add(const string &prod);
};
```

项目集`Item`类中，`prods`向量存放这个项目集的所有**项目**（即产生式+**搜索符**），而`Vn/Vt`集合存放了**所有**产生式的非终结符/终结符，`Symbol`仅仅是`Vn/Vt`的并集，为后面`GOTO`函数枚举符号提供了方便，`add`方法为项目集插入项目。这里最重要的就是重载`==`符号了，它是判断两个项目集是否相等的关键，判断也很简单，首先判断两个项目集的项目数量是否相等，若相等进一步判断是否所有的**项目**都相等，这里就展现了前面重载`==`的优点（当然还需要判断**搜索符**是否也都相等）。能判断两个项目集是否相等就好办了，后面求**项目集规范族**插入项目就简单了。

还需要注意的一点，`Prod`类已经重载`==`了，为啥项目集`prods`用向量来存，而不是直接用集合来存？这样就不用判等了，还能二分查找提高了查询效率。但是我考虑到用集合来存的话，会按照字典序来排，但是这样还要重载`<`，会打乱产生式的顺序，所以我后面的`LR`类来存放文法产生式、**项目集规范族**也是用向量来存（插入的时候判等就是了），而不是集合，以免打乱了顺序。

## `LR`类
```cpp
class LR {
	private:
		Item G; // 文法G
		enum actionStat{
			ACCEPT=0,
			SHIFT,
			REDUCE,
		};

		vector<Item> C; // 项目集规范族
		map<pair<int, char>, int> GOTO; // goto数组，项目集<int, int>=char
		map<pair<int, char>, pair<actionStat, int> > ACTION; // Action数组，Action[(i, a)]=(s|r)j
		map<char, set<char> > FIRST; // first集
		set<char> first(const string &s); // 求first集
		vector<char> inStr; // 输入串/栈
		vector<int> status; // 状态栈
		vector<char> parse; // 分析栈
		Item closure(Item I); // 求该项目的闭包
		Item Goto(const Item& I, char X); // 求I经过X到达的项目集
		void items(); // 求项目集状态机DFA！!
	public:
		void add(const string &s); // 添加产生式
		void parser(); // LR(1)分析
};
```

这是最后一个类了，重点说说。`G`用来存放输入的文法，把它看成一个项目集吧。`C`是**项目集规范族**，也就是项目集的集合，用向量来存。`actionStat`为枚举类型，用于表示`ACTION`的动作类型。`GOTO`用于存放自动机DFA上**边**，边`w(i, j)=X`，`ACTION`用于存放动作，`Action[(i, X)] = ((s|r)j)|acc`，即当状态`i`遇到**终结符X**的时候采取的动作，移进/规约/接受。`FIRST`集合存放各个**非终结符**的`FIRST`集合，`first`方法求集合会记忆化存储到`FIRST`集合里。接下来就是那三个栈了，还有`Closure`、`Goto`、`items`、`parser`这几个方法了，书上有我就不细讲了，设计好这几个数据结构，写起来会轻松很多。

## 空串处理
那时候我的分析器还不能处理形如$A\to \varepsilon$的产生式，书上、指导书给的文法，都没这类产生式。我就想，是不是`LR`不能处理空串啊？因为`LR`的`DFA`构建过程中，如果引出$\varepsilon$这条边，那就是`NFA`了，这样还要把它化为`DFA`，那就很麻烦了。请教了一下李老师，老师说处理$A\to \varepsilon$项目的时候，项目直接写成$A\to .$，也就是说求`GOTO`函数的时候不要把$\varepsilon$当终结符/非终结符处理，不要引出$\varepsilon$边。

恍然大悟，回去修改了下程序，果然能处理这类产生式了。

## `PL0`文法测试
后来有人给了一组数据，即`PL0`文法，测试失败。这里给下数据，做了点小修改，因为有些符号和我程序内部符号冲突了，所以只是做了简单的替换。
```cpp
A->B,
B->CEFH
B->H
B->CH
B->EH
B->FH
B->CFH
B->CEH
B->EFH
C->cY;
D->b=a
E->dX;
F->GB;
G->eb;
H->I
H->R
H->T
H->S
H->U
H->V
H->J
I->btL
J->fWg
K->LQL
K->hL
L->LOM
L->M
L->-M
L->+M
M->MPN
M->N
N->b
N->a
N->(L)
O->+
O->-
P->*
P->/
Q->=
Q->%
Q-><
Q->r
Q->>
Q->s
R->pKqH
S->mb
T->nKoH
U->i(X)
V->j(Z)
W->W;H
W->H
X->X,b
X->b
Y->Y,D
Y->D
Z->Z,L
Z->L
```

我调试了一下程序，发现2个比较严重的bug，有一个和程序无关紧要的bug，后面再说，这里先说下其中的一个bug，就是在求`first`集的bug。

如果文法产生式有直接左递归的话，那么就会死递归爆栈，所以我对`first`集做了下简单的修改，就是遇到直接左递归，忽略掉。

另一个`bug`也处理好了，`PL0`测试通过，近300个状态，这里贴一下局部预览图感受下。
![http://7xibui.com1.z0.glb.clouddn.com//2016/10/screenshot-window-2016-10-21-114458.png](http://7xibui.com1.z0.glb.clouddn.com//2016/10/screenshot-window-2016-10-21-114458.png)
![http://7xibui.com1.z0.glb.clouddn.com//2016/10/screenshot-window-2016-10-21-114315.png](http://7xibui.com1.z0.glb.clouddn.com//2016/10/screenshot-window-2016-10-21-114315.png)

## `for auto &` Bug
这个`bug`隐藏较深，就是我用`auto`引用类型引用容器中的元素，然后又在容器中插入元素，那么这个引用就会失效，换成迭代器也没达到预期效果，反而更糟，具体如下。

```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
	vector<int> v={1, 2, 3};
	for(const auto &i: v) {
		printf("i=%d\n", i);
		v.push_back(5);
		printf("i=%d\n", i);
	}
	return 0;
}
```

只是简单地插入一个元素而已，输出结果却是这样的。
```cpp
i=1
i=0
i=0
i=0
i=3
i=3
```

`i=0`是哪来的。。。这个没找出原因，最后简单的用`for(int i=0; ...)`来替代处理了。

现在找到原因了，看了下[cppreference.com](http://en.cppreference.com/w/)关于`push_back`的定义，有这么一句话。

> If the new size() is greater than capacity() then all iterators and references (including the past-the-end iterator) are invalidated. Otherwise only the past-the-end iterator is invalidated.

就是说如果`vector`插入元素的`size()`大于`capacity()`的话，所有迭代器、引用都会无效，否则只是最后一个元素的迭代器/引用无效。这个可能和`vector`内存分配管理有关。

然后就是打印了下`vector`的大小，发现刚开始`size()==capacity()`的，所以一插入元素就会出问题，用`reserve()`简单的把`capacity()`调大就没问题了。

## 前端处理
核心程序写完了，最后就是把它展现出来，数据格式化下就好了。

之前在验收`LL1`实验的时候，老师看到我把语法树都画出来了，就说这个实验（`LL1`）没必要画语法树，`LR1`实验能把自动机画出来就好了，然后我就爽快答应了，因为只要核心程序写出来了，那么前端随便你怎么搞都行，这部分重点说下我是怎么画自动机的。

一开始我就是用`mermaid`这个`JS`库，只要给它图的描述，它就能画出来，试了下效果不是很满意。

继续`Google`，发现这么一个工具[graphviz](http://www.graphviz.org/)，它用了一种很简单的`DOT`语言，只要用`DOT`语言来描述这个图，它就能画出来。进一步发现这个工具有`js`移植版本，就试了试，效果不错，就是现在的样子。缺点就是这个移植版本太大了，`3.5MB`的库，所以第一次加载的时间全都花在下载这个库了，其二就是画那个`PL0`自动机的时候，会因为内存不足崩溃掉，可能这是`JS`的机制问题了吧。

考虑上面2个问题，我写了一个小程序`LR_DFA`，用它在后台直接输出`DOT`文件，然后交给`graphviz`处理导出`pdf`，能把完整的`PL0`自动机（画`PL0`比较耗时）画出来，就是前面贴出来的那个图了。

