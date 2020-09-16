---
title: 如何优雅的实现C++编译期静态反射
toc: true
top: false
original: true
date: 2020-08-01 18:49:31
tags:
- C/C++
- 元编程
categories:
- 编程
---

部门请来了软件专家袁英杰咨询师指导我们软件开发，从中我也学到了很多姿势，在此记录下来宝贵的经验。苹果的mbp品控真是差劲，写这个东西把`LShift键`按坏了，真是难受。

## 反射能做什么
最近和大师聊软件设计，其中一个点是关于反射，反射最大的作用就是序列化、解序列化一个结构体，然后就能够在各个模块之间进行通信交互，不管是跨进程也好，还是跨机器也好，都缺不了反射这个功能，这也是OO世界对象交互的载体。

不然就需要人工手写一堆序列化、反序列代码，不仅代码难看，而且工作量大，容易出错。印象最深的一个例子是，大师在一个电信项目，模块之间通过TLV格式的消息进行通信，而这些TLV格式也是内部实现的，还不是标准的，然后大师定义了一套机制，只需要统一声明一次元数据的信息，然后通过`include`不同头文件，就能对同一个元数据进行不同的解释，比如序列化、解序列到数据库，序列化、解序列到网络，这也是预编译多态技术，仅用`C++98`的特性就能做到。

举一个直观一点的例子，比如打印一个结构体内容（其实就是把结构体转换成字符串）：

```cpp
struct Point {
    double x;
    double y;
};
Point p { 1, 2 };
```

那么你可能会这样写：

```cpp
printf("Point x = %d y = %d", p.x, p.y);
```

如果有成千上百个结构体，对应的打印函数（序列化到字符串）也就成千上百个，如果利用反射手段，只需要写一次，就能给所有反射对象自动生成打印函数（转换）代码。

## 引子
后来我在C++社区看到一个讨论，说`C++20`在元编程方面提供了很多便利，其中最大的遍历就是`if-constexpr`，再也不用模式匹配写一堆`enable_if`了，然后题主给了一个例子，用`C++20`的模板元求结构体的字段数量，代码如下：

```cpp
struct AnyType {
    template <typename T>
    operator T();
};

template <typename T>
consteval size_t CountMember(auto&&... Args) {
    if constexpr (! requires { T{ Args... }; }) { // (1)
        return sizeof...(Args) - 1;
    } else {
        return CountMember<T>(Args..., AnyType{}); // (2)
    }
}

int main(int argc, char** argv) {
    struct Test { int a; int b; int c; int d; };
    printf("%zu\n", CountMember<Test>());
}
```

看到这坨代码，我愣了一会，然后问大师这个求结构体字段数量是怎么做到呢？C++目前最大缺陷是缺少静态反射能力（这里指的是语言层面提供的静态反射信息，`C++23`估计会落地），应该很难做到的，分析了一会，终于看懂了，太巧妙了：

- `AnyType`声明了类型转换操作符（《C++ Modern design》书中的术语是稻草人函数），可以转换成任意类型
- 分支(2)通过不断构造所求类型`T = Test`，当无法构造时(1)，也就是输入的参数过多，这时候`参数个数 - 1`就是字段个数。

那么只能`C++20`才能做到么？这里主要用到了`C++17`的`if-constexpr`特性，`C++11`可以通过`enable-if`做到，而最主要的是那个`requires`，`C++20`才支持`concept`，`C++17`都无法做到。

然后我思考了一下，类型构造，《C++ Modern design》这本书讲过，用`sizeof`做类型推导，给的一个例子是判断一个类是否是另一个类的基类，仅通过`C++98`实现。

C++11编译期有有两大神器：`sizeof` + `decltype`，然后用这两者就能实现同样的功能，这里我用`decltype`来解决上述的`concept`问题：

```cpp
template <typename T, typename = void, typename ...Ts>
struct CountMember {
    constexpr static size_t value = sizeof...(Ts) - 1;
};

template <typename T, typename ...Ts>
struct CountMember<T, std::void_t<decltype(T{Ts{}...})>, Ts...> {
    constexpr static size_t value = CountMember<T, void, Ts..., AnyType>::value;
};

int main(int argc, char** argv) {
    struct Test { int a; int b; int c; int d; };
    printf("%zu\n", CountMember<Test>::value);
}
```

同样两种情况，用`decltype(T{Ts{}...})`来判断是否能够构造对象T。

## 如何求宏的可变参数个数？
其实这个问题价值不大，而且强依赖平凡构造函数，最大价值在后面的讨论，大师给我出了一道题，如何求宏的可变参数个数？虽然一时半会写不出来，但是之前还是看过一些框架代码的，最终实现方式如下：

```cpp
#define GET_NTH_ARG(                                                                        \
    _1,  _2,  _3,  _4,  _5,  _6,  _7,  _8,  _9,  _10, _11, _12, _13, _14, _15, _16,         \
    _17, _18, _19, _20, _21, _22, _23, _24, _25, _26, _27, _28, _29, _30, _31, _32,         \
    _33, _34, _35, _36, _37, _38, _39, _40, _41, _42, _43, _44, _45, _46, _47, _48,         \
    _49, _50, _51, _52, _53, _54, _55, _56, _57, _58, _59, _60, _61, _62, _63, _64, n, ...) n

#define GET_ARG_COUNT(...) GET_NTH_ARG(__VA_ARGS__,                     \
        64, 63, 62, 61, 60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, \
        48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, \
        32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, \
        16, 15, 14, 13, 12, 11, 10, 9,  8,  7,  6,  5,  4,  3,  2,  1)
```

`GET_ARG_COUNT(a, b, c)`展开后，会调用`GET_NTH_ARG`，然后得到`GET_NTH_ARG(a, b, c, 64, 63, ..., 3, 2, 1) 3`，从而得到最终长度3，进一步延伸，这个宏有什么作用呢？那就是对结构体进行反射，用宏提供结构体的元数据信息，从而生成一些类型信息代码。

结合之前看到的那个框架，与大师进一步交流，发现新世界，解决多年来cpp静态反射问题，一下子让很多事变成了可能。（后来找到这个实现方法的最早出处：[http://pfultz2.com/blog/2012/07/31/reflection-in-under-100-lines/](http://pfultz2.com/blog/2012/07/31/reflection-in-under-100-lines/)）

来看看大师`actor`框架中的反射例子：

```cpp
CAF_begin_message_def(Point)
    CAF_msg_field(x, double)
    CAF_msg_field(y, double)
CAF_end_message_def()

CAF_begin_message_def(Rect)
    CAF_msg_field(p1, Point)
    CAF_msg_field(p2, Point)
    CAF_msg_field(color, uint32_t)
CAF_end_message_def()
```

现在可以做到这样：

```cpp
DEFINE_STRUCT(Point,
    (double) x,
    (double) y)

DEFINE_STRUCT(Rect,
    (Point) p1,
    (Point) p2,
    (uint32_t) color)

Rect rect {
    {1.2, 3.4},
    {5.6, 7.8},
    12345678,
};
dumpObj(rect);
/* output:
{
    p1: {
        x: 1.2,
        y: 3.4,
    },
    p2: {
        x: 5.6,
        y: 7.8,
    },
    color: 12345678,
}
*/
```

后者和通常结构体定义方式非常接近，不需要再写`begin/end`原语了。完整的代码请见：

- [https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/reflection/StaticRefl.hpp](https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/reflection/StaticRefl.hpp)
- [https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/reflection/StaticRefl.cpp](https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/reflection/StaticRefl.cpp)

接下来我们一步步看看是如何实现这个的。

## 实现一套优雅的反射系统
首先来看看`Point`这个用宏声明生成展开的代码，如下：

```cpp
struct Point {
    template<typename, size_t>
    struct FIELD;
    static constexpr size_t _field_count_ = 2;
    double x;
    template<typename T>
    struct FIELD<T, 0> {
        T &obj;
        FIELD(T &obj) : obj(obj) {}
        auto value() -> decltype(auto) { return (obj.x); }
        static constexpr const char *name() { return "x"; }
    };
    double y;
    template<typename T>
    struct FIELD<T, 0 + 1> {
        T &obj;
        FIELD(T &obj) : obj(obj) {}
        auto value() -> decltype(auto) { return (obj.y); }
        static constexpr const char *name() { return "y"; }
    };
};
```

`Point`所需要的元数据都保存在`Point::FIELD<T, N>`里，而所拥有的字段数在`Point::_field_count_`中，反射只需要这两个信息，就能够生成通用的序列化、反序列化代码。

最核心的`DEFINE_STRUCT`宏定义如下：

```cpp
#define DEFINE_STRUCT(st, ...)                                                \
struct st {                                                                   \
    template <typename, size_t> struct FIELD;                                 \
    static constexpr size_t _field_count_ = GET_ARG_COUNT(__VA_ARGS__);       \
    PASTE(FOR_EACH_, GET_ARG_COUNT(__VA_ARGS__)) (FIELD_EACH, 0, __VA_ARGS__) \
};                                                                            \
```

`_field_count`信息就是由之前提到的求宏变参个数获取的：`GET_ARG_COUNT`。

接下来是`FOR_EACH`宏，作用是执行元宏`FIELD_EACH`一定次数（字段数量），而`FIELD_EACH`接收两个参数：

- 当前字段`id`（用于生成`FIELD<T, id>`）
- 当前字段声明信息，例如`(double) x`（用于定义`double x`，并实现`FIELD<T, id>`内容）

先来看看`FIELD_EACH`的定义：

```cpp
#define FIELD_EACH(i, arg)                    \
    PAIR(arg);                                \
    template <typename T>                     \
    struct FIELD<T, i> {                      \
        T& obj;                               \
        FIELD(T& obj): obj(obj) {}            \
        auto value() -> decltype(auto) {      \
            return (obj.STRIP(arg));          \
        }                                     \
        static constexpr const char* name() { \
            return STRING(STRIP(arg));        \
        }                                     \
    }                                         \
```

`PAIR(arg)`这个宏比较有意思，定义如下：

```cpp
#define PAIR(x) PARE x // PAIR((double) x) => PARE(double) x => double x
#define PARE(...) __VA_ARGS__
```

举个例子，`PAIR((double) x)`会展开成`PARE(double) x`，因为`PARE(double)`得到的是`double`，所以最终结果是`double x`，从而定义字段。

另一个关键点是`return (obj.STRIP(arg))`，看看`STRIP`实现：

```cpp
#define STRIP(x) EAT x // STRIP((double) x) => EAT(double) x => x
#define EAT(...)
```

同样的例子，`STRIP((double) x)`会展开成`EAT(double) x`，而`EAT(double)`得到空结果，所以最终结果是`return (obj.x);`，这样就能通过`value()`函数拿到成员字段的**引用**。

而最后一个`STRING(STRIP(arg))`就比较简单了，通过`STRING`得到对应参数字符串，宏的基本用法了：

```cpp
#define STRING(x) STR(x)
#define STR(x) #x
```

`PASTE(FOR_EACH_, GET_ARG_COUNT(__VA_ARGS__))`这句是为了拼出`FOR_EACH_N`，`PASTE`实现如下：

```cpp
#define PASTE(x, y) CONCATE(x, y)
#define CONCATE(x, y) x ## y
```

比如这个例子最终会展开成`FOR_EACH_2(FIELD_EACH, 0, (double) x, (double) y)`，继续看看`FOR_EACH_2`定义：

```cpp
#define FOR_EACH_1(func, i, arg)        func(i, arg);
#define FOR_EACH_2(func, i, arg, ...)   func(i, arg); FOR_EACH_1(func, i + 1, __VA_ARGS__)
...
```

也很简单，直接看展开结果吧：

```cpp
FIELD_EACH(0, (double) x);
FIELD_EACH(0 + 1, (double) y);
```

最后这一切，通过宏展开拼装在一起，从而得到所有元信息代码。

## 遍历查询反射信息
有了宏的一臂之力，接下来就是模板元编程发挥威力的地方了，首先我们需要定义一个高阶函数`forEach`，实现*Vistor模式*，其接受两个参数：

- 传递反射的对象`T&& obj`
- 一个函数`f`，对对象各个字段进行访问、操作，签名为`void(const char* fieldName, FieldT& value)`

```cpp
template<typename T, typename F, size_t... Is> // (1)
inline constexpr void forEach(T&& obj, F&& f, std::index_sequence<Is...>) {
    using TDECAY = std::decay_t<T>;
    (void(f(typename TDECAY::template FIELD<TDECAY, Is>(obj).name(),
            typename TDECAY::template FIELD<TDECAY, Is>(obj).value())), ...);
}

template<typename T, typename F> // (2)
inline constexpr void forEach(T&& obj, F&& f) {
    forEach(std::forward<T>(obj),
            std::forward<F>(f),
            std::make_index_sequence<std::decay_t<T>::_field_count_>{});
}
```

先看版本(1)，通过参数包`Is...`展开代码，从而将函数`f` `apply`到各个参数上，还是以`Point`为例，展开代码如下：

```cpp
(f(Point::FIELD<0>(obj).name(), Point::FIELD<0>(obj).value()),
f(Point::FIELD<1>(obj).name(), Point::FIELD<1>(obj).value()));
```

而`std::index_sequence<0, 1>`可以通过`std::make_index_sequence`得到，避免用户指定字段个数，这也是最终版本(2)所做的事。

## 反射系统的应用
最后来看看反射最基本的一个应用，也就是序列化，将结构体序列化成字符串，从而打印出来，我们通过实现`dumpObj`来做到：

```cpp
template<typename T>
void dumpObj(T&& obj, const char* fieldName = "", int depth = 0) {
    auto indent = [depth] {
        for (int i = 0; i < depth; ++i) {
            std::cout << "    ";
        }
    };

    if constexpr(std::is_class_v<std::decay_t<T>>) { // (1)
        indent();
        std::cout << fieldName << (*fieldName ? ": {" : "{") << std::endl;
        forEach(obj, [depth](auto&& fieldName, auto&& value) {
            dumpObj(value, fieldName, depth + 1);
        });
        indent();
        std::cout << "}" << (depth == 0 ? "" : ",") << std::endl;
    } else { // (2)
        indent();
        std::cout << fieldName << ": " << obj << "," << std::endl;
    }
}
```

这是一个递归版本，通过检查是否为基本类型，来判断是否需要递归打印。如果是基本类型，走分支(2)，直接将其打印出来，如果是结构体，走分支(1)，进一步递归遍历结构体各个字段，直到基本类型为止。

## 结论、展望未来
可惜目前C++语言未能提供反射信息，目前只能手动描述对应的元信息，综上是C++反射的优雅实现，仅需要实现一遍，通过宏展开生成代码，结合模板元编程的威力，就能为任意结构体生成对应的序列化、反序列化代码，减少程序员重复劳动、容易出错的问题。

期待未来`C++23`能提供反射信息，利用其模板元生成局部代码来替代宏，将减少这些`tricky`代码，不过目前该方案已经足够好用。

`Rust`的宏能力也很强，能够匹配`{`, `[`, `(`，而C的宏只能匹配`(`，最后定义出来的语句就不够直观了。在C++中需要结合宏与模板元来生成代码，而Rust只需要*过程宏/属性宏*和*声明宏*，统一的宏机制就能达到类似效果，而且宏内还能做模式匹配。若`C++23`的模板能够生成局部代码，那么也能统一用模板机制搞定很多事了。

最后引用大师说的，`C++`就像一片大海，给程序员足够的自由，挖掘的越多，乐趣也就越多，每个点都能挖掘出来很多玩法，比如宏，比如模板。世界是复杂的，需要一门大而全的语言来应对这一切复杂。

