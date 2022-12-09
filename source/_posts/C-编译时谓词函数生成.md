---
title: C++编译时谓词函数生成
toc: true
original: true
date: 2022-12-09 19:14:53
tags:
- C/C++
- 元编程
categories:
- 编程
---
## 背景
谓词函数是返回类型为`bool`的函数，这类函数在编程中非常常见，例如STL标准库中的`std::find_if/std::sort`等函数就要求接受一个谓词。

这类谓词可以通过取反、与、或等操作符组合成更高级的谓词，考虑如下场景：

```cpp
bool p1(int, int);
bool p2(int, int);
```

如果要求对谓词`p1`取反，不难写出如下代码：

```cpp
bool not_p1(int x, int y) { return !p1(x, y); }
```

要求`p1`与`p2`进行与和或组合，不难写出如下代码：

```cpp
bool p1_and_p2(int x, int y) {
    return p1(x, y) && p2(x, y);
}

bool p1_or_p2(int x, int y) {
    return p1(x, y) || p2(x, y);
}
```

要求程序员手写这类组合性的代码，那是多么无趣的一件事情，如果谓词入参一多、谓词组合数量一多，那么也容易出错，能否将这件事情交给机器解决呢？

通过C++元编程技术，可以实现如下效果：

```cpp
constexpr auto not_p1 = notP<p1>;
constexpr auto p1_and_p2 = andP<p1, p2>;
constexpr auto p1_or_p2 = orP<p1, p2>;
```

甚至更复杂的逻辑组合：

```cpp
constexpr auto complex_p = andP<orP<p1, p2, p3>, notP<p4>, p5>;
// 对应手写代码如下
bool complex_p(int x, int y) {
    return (p1(x, y) || p2(x, y) || p3(x, y)) && !p4(x, y) && p5(x, y);
}
```

其中`andP/orP/notP`就是我们所需要的组合子，它只需要接受同签名的谓词函数，便能按照逻辑与或非的语义对其进行组合，并生成组合后的函数，而这一切都发生在编译时，与手写代码相比无任何损失。

## 实现
第一个问题是如何获取传入的谓词的签名，以便提取入参呢？这可以通过类模板+偏特化实现：

```cpp
template<typename Pred>
class LogicalPredGenerator;

template<typename ...Ts> // 偏特化
class LogicalPredGenerator<bool(*)(Ts...)>
{
    using Pred = bool(*)(Ts...);
};
```

其中`LogicalPredGenerator<decltype(p)>::Pred`便能拿到谓词的签名，以及每个入参参数包`Ts`。

接下来实现3个组合子就简单了，通过可变参数实现：

```cpp
template<typename ...Ts>
class LogicalPredGenerator<bool(*)(Ts...)>
{
    using Pred = bool(*)(Ts...);
public:
    template<Pred p>
    static bool notP(Ts... ts) {
        return !p(ts...);
    }
    template<Pred p1, Pred p2, Pred... pN>
    static bool andP(Ts... ts) {
        return p1(ts...) && p2(ts...) && (pN(ts...) && ...);
    }
    template<Pred p1, Pred p2, Pred... pN>
    static bool orP(Ts... ts) {
        return p1(ts...) || p2(ts...) || (pN(ts...) || ...);
    }
};
```

最后用变量模板封装一下接口：

```cpp
template<auto p>
constexpr auto notP = LogicalPredGenerator<decltype(p)>::template notP<p>;

template<auto p1, auto... pN>
constexpr auto andP = LogicalPredGenerator<decltype(p1)>::template andP<p1, pN...>;

template<auto p1, auto... pN>
constexpr auto orP = LogicalPredGenerator<decltype(p1)>::template orP<p1, pN...>;
```
