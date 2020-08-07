---
title: 《Effective Modern C++》第一章笔记：Deducing Types
toc: true
top: false
original: true
date: 2017-05-12 09:25:59
tags:
- C/C++
categories:
- 学习笔记
---

这几天人工智能成绩出来了，还是开卷考试，竟然挂掉了，真是意不意外。。。请导员查了一下平时分：卷面46/100，平时53/100，按经验来说，只要卷面过40，靠平时分还是能拉到及格的，以前平时分一般会给到90左右，这次真是吃了平时分的亏了。

回忆了一下，人工智能这门课不点名，但是每节课都会有随堂测试，靠这个来算平时分吧。这门课我印象中缺过两节课，因为前段时间在忙比赛，加上开卷，期间也和老师提前打过招呼（邮件没回复，估计没看到），随堂测试靠度娘。结果居然因为平时分太低也救不了了。

最近买了本动物书《Effective Modern C++》，好评，从头到尾指出了一些C++11/14的坑，同时介绍一些技巧如何避免这些坑。

这里做一下笔记，备忘。我用的是`clang`编译器。

这章主要是介绍C++11/14的推导规则，以及各种坑。

## Item 1: Understand template type deduction
首先从模板类型推导开始，后面的`auto`关键字和模板类型推导规则差不多的。需要注意下面说的`parameter`就是形参，而`argument`是实参。

首先用如下伪代码介绍：
```cpp
template<class T>
void f(ParamType param); // 模板函数

f(expr); // 调用f，argument是expr，根据expr来确定T和param的类型。
```
其中T的类型不仅由expr来确定，还由ParamType来确定的。这里有3种情况：

- ParamType是一个指针或者引用，但不是通用引用。（这里可以先理解为`T &&`。）
- ParamType是通用引用。
- ParamType既不是引用也不是指针

### Case 1: ParamType是指针或者引用，但不是通用引用。
规则如下：
1. 如果expr是引用，那就忽略其引用部分
2. 根据expr的类型来匹配ParamType和T

### Case 2: ParamType是通用引用
规则如下：
- 如果expr是左值，那么T和ParamType都被推导成左值引用
- 如果expr是右值，规则和Case 1一样。即忽略引用部分（&&），然后根据expr的类型来匹配ParamType和T

### Case 3: ParamType既不是引用也不是指针
既然ParamType既不是引用也不是指针，那么用的就是传值方式了。

规则如下：
1. 如果expr是引用，那么忽略引用部分
2. 忽略expr的引用部分后，如果带`const/volatile`修饰符，那么也忽略掉
3. 最后根据expr的类型来匹配ParamType和T

举个例子：
```cpp
template<class T>
void f(T param); // 模板函数

int x = 233;
const int &rx = x;
f(rx); // T和param的类型为int
```

这里的rx虽然为`const int &`类型，根据规则1和2，忽略引用和const，那么就变成了`int`类型。之所以要去掉const，是因为expr不能被修改并不意味着其副本不能被修改。也就是说最后的param可以修改，因为修改它不会影响到rx。

还有一种情况类似，需要注意。
```cpp
const char *const ptr = "hello Netcan";

f(ptr); // T和param的类型为const char *
```

根据规则2，去掉const限定符，那么变成的是`const char*`类型。可以看出去掉的是后面那个const，因为ptr的值是const，表明ptr不能被修改，传值的时候可以去掉；而前面那个const表面ptr**指向的内容**不能被修改，限定的不是ptr，需要保留的。

### 数组作为参数
这种情况比较特殊，毕竟数组类型和指针不同，尽管有时候可以看成是同一个东西。之所以引起大多数人的疑惑，是因为在许多情况下，数组会退化(decay)成指针（指向它的第一个元素），这种退化才能允许如下代码通过：
```cpp
const char name[] = "Netcan";
const char * ptrToName = name; // 这种情况就是数组退化成指针了
```

这里`ptrToName`是一个`const char*`指针，指向的是数组name，而数组name的类型是`const char[7]`，显然`const char*`和`const char[7]`并不是同一个类型，但是因为数组的退化规则，导致代码能编译通过。

如果数组传给一个模板（Case 3），情况如何？
```cpp
f(name); // T被推导成const char*/const char[]
```

很显然，T被推导成一个指针类型了，并不是真正的数组类型。但是如果声明成引用参数，如下：
```cpp
template<class T>
void f(T& param); // 模板函数
f(name);
```

这里T被推导成真正的数组类型了，也就是说，T被推导成`const char[7]`，而param的类型为`const char(&)[7]`。

根据这个技巧，可以写出如下模板函数，推导出数组的元素个数。
```cpp
template<class T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept { // 因为我们关心的是数组元素个数，所以参数名可以省略了
	return N;
}
```

constexpr关键字表明在编译过程中就能确定出结果，noexcept有助于编译器优化。

书上推荐用内置的`std::array`类型作为数组使用。

### 函数作为参数
不仅仅是数组类型能退化成指针，函数类型也能退化成函数指针。这种情况和前面说的数组推导是一样的。这种情况在实践中用引用和指针没什么区别，不再概述，原理懂了就行。

### Things to Remember
- 模板推导中，argument的引用性可以忽略
- 当param是通用引用的时候，argument为左值需要特别注意
- 当param是传值类型(Case 3)的时候，argument的const/volatile需要忽略
- 模板推导中，argument为数组或者函数的时候会退化成指针，除非param用引用

## Item 2: Understand auto type deduction
auto类型推导和模板推导差不多，但有些地方不同。之前说的模板参数推导：
```cpp
template<class T>
void f(ParamType param); // 模板函数

f(expr); // 调用f，argument是expr，根据expr来确定T和param的类型。
```

当用auto来声明变量的时候，auto相当于这里面的T，而变量类型相当于ParamType。所以也有三种情况：
- Case 1: 声明的类型是指针或者引用，但不是通用引用
- Case 2: 声明的类型是是通用引用
- Case 3: 声明的类型既不是引用也不是指针

按照模板推导的方法来做就是了。

但是有一点不同，需要特别指出。

C++98有两种初始化变量的方法：
```cpp
int x1 = 27;
int x2(27);
```

C++11添加了**通用**初始化（就是花括号）变量的方法，如下：
```cpp
int x3 = {27};
int x4{27};
```

后面Item 5会提到利用auto来代替直接指出变量类型的优点，所以将int用auto替代得到如下代码：
```cpp
auto x1 = 27;
auto x2(27);
auto x3 = {27};
auto x4{27};
```

如上声明都能通过编译，但是有区别。前两条语句都是将x声明为int类型，而**后两条**语句声明的是`std::initializer_list<int>`类型，其包含了一个元素27，这是一个巨大的陷阱，也是一些程序员只有在必须要用到花括号初始化的时候才用的原因。（需要注意的是，由于C++标准委员会接受了提议`N3922`，这个提议使得x4这种初始化不再为`std::initializer_list`类型，而是`int`类型了。由于`N3922`不是C++11和C++14的部分，所以有些编译器实现了`N3922`，比如说我用的`clang`就是把x4推导为`int`类型。）

这就是auto类型推导需要**特别注意**的地方。当auto声明的变量初始化用花括号的时候，推导的类型为`std::initializer_list`。如果无法推导，那么报错。

而模板推导的时候，若传通用初始化，将会**报错**，这点和auto不同，它不会推导成`std::initializer_list`类型。

关于通用初始化，这两者表现出来的行为不同，作者也无法理解，就把它当做规则来记吧。

然而在C++14中，允许auto作为函数**返回类型**，用来推导函数的返回类型，并且C++14也支持在lambdas中使用auto来声明**param**。这个时候auto使用的是**模板推导**的规则，而不是auto的推导规则了。也就是说如下代码无法编译通过：

```cpp
auto createInitList() {
	return {1, 2, 3}; // 编译错误！无法推导出{1, 2, 3}的类型
}

std::vector<int> v;
auto resetV = [&v](const auto& newValue) { v = newValue; };

resetV({1, 2, 3}); // 编译错误！理由同上
```
### Things to Remember
- auto类型推导通常和模板类型推导规则一样，但是auto类型推导将花括号初始化看做`std::initializer_list`，而模板类型推导不会
- auto用在函数返回值或者lambdas的param中，用的是模板推导的规则而不是auto类型推导规则。

## Item 3: Understand decltype
decltype关键字用来给出一个变量名或者表达式的**真正**类型。通常情况下使用没什么问题，这里不在举例。

在C++11中一个基本的用法就是用在声明函数的返回类型，而返回类型依赖于函数的param类型。

一般来说`operator[]`返回的通常是容器中元素的引用，即`T&`。也有例外，比如`std::vector<bool>`返回的不是`bool&`，而是一个新的临时对象，这点非常重要，在Item 6中会提到。

假如现在需要写一个函数，对用户进行权限检查，然后再索引容器中的元素，如下：
```cpp
template<class Container, class Index>
?type? authAndAccess(Container& c, Index i) {
	autheticateUser();
	return c[i];
}
```

那么这个`?type?`应该是什么类型呢？无从判断，这时候decltype派上用场了：
```cpp
template<class Container, class Index>
auto authAndAccess(Container& c, Index i) -> decltype(c[i]) {
	autheticateUser();
	return c[i];
}
```

注意这里的auto并没别的作用，只是占位符罢了。而param表后面的`->`是C++11的*trailing return type*语法，因为decltype需要用到前面声明的c和i。

在C++14中可以省略后置返回类型写法，比如前面提到的，只用auto来推导函数的返回类型，将使用模板推导的规则：
```cpp
template<class Container, class Index>
auto authAndAccess(Container& c, Index i) {
	autheticateUser();
	return c[i];
}
```

细心的话，会发现这样写有严重的bug，那就是**丢失引用**。因为c[i]返回的是`T&`类型，而函数用的是`auto`，根据模板推导规则的Case 3，会导致函数返回的类型为`T`，变成了右值。

上面这个例子就是想说明，我们需要decltype这样的类型推导，因为它能返回**精确**的类型。

C++预料到了这种情况，在C++14中提出了`decltype(auto)`说明符，很好地解决了这个问题，因此可以改写成这样：
```cpp
template<class Container, class Index>
decltype(auto) authAndAccess(Container& c, Index i) {
	autheticateUser();
	return c[i];
}
```

这时候`authAndAccess`将会返回和c[i]一样的类型了，比如c[i]返回的是`T&`，那么`authAndAccess`也是`T&`；c[i]返回的是对象，那么`authAndAccess`也将返回对象。

同时也能用在初始化表达式中，如下：
```cpp
Widget w;
const Widget &cw = w;
auto myWidget1 = cw; // myWidget1的类型为Widget
decltype(auto) myWidget2 = cw; // myWidget2的类型为const Widget &
```

`authAndAccess`还有一个问题，它的argument是一个左值引用，非const，因为程序调用它可能需要修改检索到的值，这没什么问题。但是不能传一个右值argument给它，因为右值不能绑定到左值引用上。一个方案是重载`authAndAccess`，写一个左值和右值版本的，这样就需要维护两个版本了。因此可以用通用引用来实现，得出如下版本：
```cpp
template<class Container, class Index>
decltype(auto) authAndAccess(Container&& c, Index i) {
	autheticateUser();
	return std::forward<Container>(c)[i];
}
```

decltype也有例外情况，这里需要**注意**下。C++定义左值表达式`(x)`为一个左值，所以`decltype((x))`的类型将是一个左值引用。也就是说：
```
int x = 2;
decltype((x)) y = x; // y的类型为int &

decltype(auto) f1() {
	int x = 0;
	return x; // 返回值为int，因为decltype(x)为int
}

decltype(auto) f2() {
	int x = 0;
	return (x); // 返回值为int&，因为decltype((x))为int&，返回一个局部变量的引用，非常危险
}
```
### Things to Remember
- decltype大多数时候返回的类型和变量或者表达式的类型一致
- 左值表达式(x)返回的是它的引用类型
- C++14支持decltype(auto)说明符，虽然含有auto，实际上用的是decltype的规则。

## Item 4: Know how to view deduced types
这一节讲了如何查看C++推导出来的类型。

### IDE编辑器
这个方法比较简单，需要IDE支持（IDE从编译器那里取得相关信息），比如将鼠标放到变量上，就会显示出该变量的类型了。

但是也有缺点，就是当涉及到比较复杂的类型，那么给出来的信息就没什么用了。
### 编译器诊断
更好的办法是用编译器编译的时候给出类型信息，就是让编译器报错，从而给出相关信息。

所以可以这么做，声明一个未定义的模板：
```cpp
template<class T>
class TD;
```

当定义一个TD对象的时候，由于TD未定义，所以会报错。比如我想知道x的类型，那么：

	TD<decltype(x)> xType;

编译得出如下信息：

	error: implicit instantiation of undefined template 'TD<const int *>'
        TD<decltype(x)> xType;

很容易看出，x的类型为`const int *`。

### 运行时输出
如果想让程序运行过程中，给出类型信息，要怎么做呢？书上首先介绍了`typeid`操作符，`typeid`会返回一个`std::type_info`对象，它有一个成员函数`name`，可以返回类型的字符串形式(`const char*`)。

```cpp
cout << typeid(x).name << endl;
```

输出的结果为`PKi`，`PK`表示`const *`，`i`表示`int`。可以用`c++-filt`这个工具来获得可读的信息：
```
$ ./a.out | c++filt -t
  int const*
```

但是`typeid`也有巨坑，这里给出一个比较复杂的例子：
```cpp
template<class T>
void f(const T& param);

const auto vw = vector<Widget>(...); // ...省略初始化的元素了
f(&vw[0]);
```

那么这个T和param的类型是什么呢？先来分析下，`vw`是`const vector<Widget>`类型，那么`vw[0]`就是`const Widget &`类型，`&vw[0]`是`const Widget *`类型。然后根据param匹配，很容易知道param的类型为`const Widget * const &`了，从而得知T的类型为`const Widget *`。

那么看看`typeid`给出来的结果：
```cpp
template<class T>
void f(const T& param) {
	using std::cout;
	cout << "T = " << typeid(T).name() << endl;
	cout << "param = " << typeid(param).name() << endl;
}
```

编译运行：

```bash
$ ./a.out | c++-filt -t
  T = Widget const*
  param = Widget const*
```

可以看出T的类型对的，而param的类型是错的！再说了T怎么可能和param同一个类型，这也说明了`std::type_info::name`给出来的信息不可靠，因为`typeid`是通过传值方式（即Item 1的Case 3）来处理参数，将param的引用和const忽略后，从而变成`T param`，这也就是为啥T和param的类型一样了。

具体来说：
```cpp
printf("%d\n", typeid(const int) == typeid(int)); // true
printf("%d\n", typeid(int &) == typeid(int)); // true
```

如果想得到正确的信息，最后给出的解决方案是用*Boost.TypeIndex*。
```cpp
#include <boost/type_index.hpp>
template<class T>
void f(const T& param) {
	using std::cout;
	using boost::typeindex::type_id_with_cvr;

	cout << "T = "
		<< type_id_with_cvr<T>().pretty_name()
		<< endl;
	cout << "param = "
		<< type_id_with_cvr<decltype(param)>().pretty_name()
		<< endl;
}
```

得出如下结果：
```cpp
T = Widget const*
param = Widget const* const&
```

顾名思义，`with_cvr`就表示保留`const/volatile`和引用了。

### Things to Remember
- 查看推导的类型可以通过IDE，编译器错误信息，*Boost.TypeIndex*库来实现
- 最重要的是理解C++的推导规则
