---
title: 《Effective Modern C++》第二章笔记：auto
toc: true
top: false
original: true
date: 2017-05-14 15:19:26
tags:
- C/C++
categories:
- 学习笔记
---

这章更加详细地介绍auto关键字。

## Item 5: 优先考虑使用auto来代替显式类型声明
习惯C的写法可能都会为变量指出一个类型，然而这有可能忘记初始化；有些场合若显式指出变量的类型，那么写起来非常长；将局部变量定义成闭包类型，因为闭包类型只有编译器知道，所以无法显式指出它的类型。

因为C++11有了auto关键字，这些问题很好地解决了。首先用auto来定义一个变量的话，必须要初始化，这就避免了变量未初始化的错误；其次能节省输入时间；auto类型推导也可以指出只有编译器才知道的类型（比如说闭包）。

```cpp
auto derefUpless =
	[](const std::unique_ptr<Widget> &p1,
	   const std::unique_ptr<Widget> &p2)
	{ return *p1 < *p2; };
```

在C++14中甚至能省略参数的类型：
```cpp
auto derefUpless =
	[](const auto &p1,
	   const auto &p2)
	{ return *p1 < *p2; };
```

关于用局部变量来存储闭包，可能会优先考虑`std::function`函数对象，因为函数对象也能存储闭包。比如上面那个闭包，可以这样写：

```cpp
std::function<bool(const std::unique_ptr<Widget> &,
				const std::unique_ptr<Widget> &)>
	derefUpless = [](const std::unique_ptr<Widget> &p1,
				const std::unique_ptr<Widget> &p2)
				{ return *p1 < *p2; };
```

首先可以看出，参数太冗长了，param都要写两遍。`std::function`和使用auto定义的闭包并不相同。auto定义的闭包的变量大小和闭包本身一样大，而`std::function`定义的对象来保存闭包都有固定的大小，当闭包过大的时候，它会分配堆空间来存储，有可能产生内存溢出错误，这导致在通常情况下`std::function`对象用的内存比auto定义的要多；也由于是间接调用闭包，这也会比auto定义的要慢。

虽然有很多理由使用auto，同时也能避免显式声明的一些坑，但auto也有坑，有时候用auto推导出来的类型可能并不是自己想要的，这在Item 2到Item 6介绍过了。

### Things to Remember
- auto定义的变量必须初始化，能避免一些类型不匹配导致的移植或者性能问题；同时也有利于重构，减少显式声明的代码量。
- auto也有坑，在Item 2到6介绍过了。

## Item 6: 当auto推导出不想要的类型时，使用显式类型声明
这里介绍了一种情况，使用`std::vector<bool>::operator[]`来检索元素的时候。比如说：
```cpp
vector<bool> features(const Widget &w); // 返回widget对象的属性位向量

Widget w;

bool highPriority = features(w)[5];

process(highPriority);
```
如果把上面的`bool`换成auto来声明的话，那么就会出现未定义行为了。

因为`vector<bool>::operator[]`返回的并不是`bool &`，因为C++是按**字节**寻址的，不可能返回一个比特位的地址。为了达到和`bool &`一样的效果，它返回的是一个`vector<bool>::reference`对象，可以隐式转换成`bool`类型。

再来看看使用显式声明的`bool`定义为什么没有问题，因为`operator[]`返回一个`vector<bool>::reference`对象，然后转换成`bool`，赋值给`highPriority`，这相当于取第5位。

而使用auto声明的`highPriority`，将是一个`vector<bool>::reference`类型的对象。它的值取决于实现，一种实现是`reference`指向内存中包含那一位的字，然后通过位移运算来取得指定位。

因为`features`这个函数返回的是`w`的属性位向量，是一个**临时变量**。当赋值语句结束后，临时变量就销毁了，到后面的`process(highPriority)`的时候，`highPriority`包含悬空指针，导致不确定的行为。

`vector<bool>::reference`也是代理类(*proxy class*)的一个例子，它为对象提供一种代理以控制对这个对象的访问。在某些情况下，一个客户不想或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。不过`vector<bool>::operator[]`的这个代理类隐藏的比较深，几乎看不出来被代理了。而C++有些代理类设计的比较明显，比如`std::shared_ptr`和`std::unique_ptr`。

对于这些隐藏的代理类，就不要直接用auto了，因为代理类的对象一般活不长。

当然这并不是auto的错，只是它没有推导出想要的类型，可以通过显式类型定义或者强制类型转换来解决。比如说：

	auto highPriority = static_cast<bool>(features(w)[5]);

也并不是代理类这么干，比如auto推导出自己不想要的类型，可以通过强制类型转换来得到。

### Things to Remember
- 隐藏的代理类可能使得auto推导出一个错误的类型
- 使用强制类型转换来使得auto推导出想要的类型


