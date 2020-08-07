---
title: 《Effective Modern C++》第三章笔记：Moving to Modern C++
toc: true
top: false
original: true
date: 2017-07-22 15:49:17
tags:
- C/C++
categories:
- 学习笔记
---
这一章介绍了C++11/14的最新的基本特性。

## Item 7: 使用()和{}创建对象的区别
C++有以下几种初始化对象的方法：

	int x(0); // 圆括号初始化
	int y = 0; // 等于号初始化
	int z {0}; // 花括号初始化
	int z = {0}; // 同上，和花括号初始化一样

需要注意的是，自定义类使用`=`的时候，可能会调用拷贝构造函数或者赋值(`operator=`)函数。

在C++98中，无法为STL容器初始化多个值；而在C++11中，引进一个叫做通用初始化*uniform initialization*的东西，这个东西基于`{}`初始化，可以用在很多地方，所以也称为花括号初始化*braced initialization*。

于是，可以这么表达初始化多个值：

	vector<int> {1, 2, 3};

在C++11中，可以为类内非静态成员初始化默认值，用`=`和`{}`都可以，但是用`()`不行：

	class Widget {
		private:
			int x{0}; // 可以这么初始化
			int y = 0; // 同上
			int z(0); // 不能这么初始化
	}

而对一些不可复制的对象（如`std::atomics`）就不能使用`=`来初始化了，而`()`和`{}`没问题。

通过以上例子说明了为啥`{}`为通用初始化了，C++有3种初始化方法，而`{}`都能用上。

`{}`还有一个特性就是，不能进行隐式窄类型转换，否则编译器会报错；然而`=`因为需要兼容老代码所以不会报错。

	double x, y, z;
	int sum1{x + y}; // error: type 'double' cannot be narrowed to 'int' in initializer list [-Wc++11-narrowing]
	int sum2 = x + y; // 没问题
	int sum3(x + y); // 没问题

还有一个问题，使用`()`初始化可能会产生混淆，例如创建一个无参默认对象的时候：

	Widget w2(); // 声明一个w2返回Widget的函数

这时候使用`{}`初始化，就不会出现这个问题了：

	Widget w3{}; // 创建一个w3对象

综上所述，`{}`初始化有以下优点：

- 使用场景多
- 防止隐式窄类型转换
- 防止混淆

但是`{}`也有缺点的，主要是和`initializer_list`与构造函数重载有关。前面Item 2也说过了，auto类型推导会将花括号初始化看做`std::initializer_list`类型。

当构造函数没有使用`initializer_list`的时候，使用`()`和`{}`初始化是一样的。若重载构造函数使用了`initializer_list`的时候，使用`{}`初始化，编译器会尽可能使用`initializer_list`版本的构造函数，哪怕其他构造函数更加合适（不过书上也给出了例外情况）。

若既有默认构造函数，又有`initializer_list`版本的构造函数，那么当使用`{}`不带参数初始化的时候，将使用的是默认构造函数而不是`initializer_list`版本的了。如果想要表达的是，默认值是空的`initializer_list`的话，就用`({})`或者`{ {} }`来初始化。

### Things to Remember
- `{}`初始化应用范围广
- 在重载构造函数中，`{}`初始化会尽可能使用`initializer_list`版本的构造函数，哪怕其他构造函数更加合适
- 创建`vector`对象的时候，使用`()`和`{}`初始化的作用各不相同。
- 在模板中创建对象，使用`()`还是`{}`初始化需要仔细考虑。

## Item 8: 优先考虑使用nullptr代替0或者NULL
因为0在C++中是一个int类型，而不是一个指针类型，只有在指针环境下才会把0看做空指针。同样NULL也不是指针类型，而是数值类型（可能是int也可能是long，依赖于实现），也就是说，0和NULL都不是指针类型。

假如有一下重载函数

	void f(int);
	void f(bool);
	void f(void*);

	f(0); // 调用的是f(int)，而不是f(void*)
	f(NULL); // 可能会有歧义，因为NULL可能是long也可能是int，假设是long，那么既可以转换为int，也可以转换为bool，还可以转换为void*

也就是说，我想调用的是`f(void*)`，然而实际调用的确实数值类型的。所以程序员一般避免重载指针类型和数值类型，其实当nullptr出现后，这种情况就可以解决了，而且也可以使得代码更加清晰。

nullptr的优势是它不是一个数值类型，其实也不是指针类型，但是可以把它看做是所有类型的指针。因为nullptr是std::nullptr_t类型的实例，而std::nullptr_t类型可以隐式转换成指针类型。

这在模板中非常有用，因为模板会推导类型，如果使用0或者NULL的话，将会被推导成数值类型，那么有可能就无法初始化指针了。
### Things to Remember
- 优先考虑使用nullptr代替0或者NULL
- 避免重载指针类型和数值类型

## Item 9: 优先考虑使用using代替typedef声明
在C++98中可以使用typedef为一个类型起名：

	typedef
		std::unique_ptr<std::unordered_map<std::string, std::string>>
		UPtrMapSS;

C++11中提供了一个别名声明*alias declarations*，即使用using：

	using UPtrMapSS =
		std::unique_ptr<std::unordered_map<std::string, std::string>>;

这两个都能达到一样的效果，那么using有什么优势呢？使用using会比较清晰：

	typdef void (*FP)(int, const std::string&); // FP为函数指针
	using FP = void(*)(int, const std::string&); // 同上

其实还有比较重要的一点就是，using可以用来作为模板别名(*alias templates*)，而typedef不行。

	templates<class T>
	using MyAllocList = std::list<T, MyAlloc<T>>; // MyAllocList<T>就是std::list<T, MyAlloc<T>>

如果要使用typedef来达到同样效果，那么可以外套一个结构体：

	template<class T>
	struct MyAllocList {
		typdef std::list<T, MyAlloc<T>> typde;
	}; // MyAllocList<T>::type就是std::list<T, MyAlloc<T>>

如果我想在一个模板类内使用`MyAllocList<T>::type`链表来存储数据的话，那么情况更复杂：

	template<class T>
	class Widget {
		typename MyAllocList<T>::type list; // 需要使用typdename前缀
		...
	};

`MyAllocList<T>::type`表明是一个依赖模板参数的类型(*dependent type*)，必须要加一个`typdename`前缀来表明它是一个依赖类型（否则编译器无法区分`::type`是类型还是数据成员）。

如果使用using别名声明的话，就不用那么麻烦了：

	template<class T>
	class Widget {
		MyAllocList<T> list; // 省略了typename前缀，也省略了::type后缀
		...
	};

也因为using表明`MyAllocList<T>`就是一个类型的别名，所以无需`typdename`来说明。

虽然C++11就支持using了，但是这里有必要提一个东西，那就是STL的*type traits transformation*(`std::transformation<T>::type`)，可以给类型加修饰符的东西，但是它们是用typedef来搞的，例如

	template<class T> struct remove_const<const T> {typedef T type;}; // 去掉const的修饰符，const T -> T
	template<class T> struct remove_reference<T&> {typedef T type;}; // 去掉引用，T& -> T

也就是说，如果在模板中使用这些东西，需要加`typdename`前缀。由于历史原因，直到C++14，才使用using来定义这些*type traits transformation*(`std::transformation_t<T>`)：

	std::remove_const<T>::type // C++11: const T -> T
	std::remove_const_t<T> // 同上

将typedef的类型转换成用using定义，也很简单：

	template<class T>
	using remove_const_t = typename remove_reference<T>::type;

### Things to Remember
- typedef不支持模板化，而using别名声明可以
- 使用using别名声明可以避免`::type`后缀，在模板中，能避免typename前缀
- C++14提供using版本的*type traits transformation*来代替C++11的typedef版本


