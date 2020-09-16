---
title: C++元编程之Parser Combinator
toc: true
original: true
date: 2020-09-16 18:26:25
tags:
- C/C++
- 元编程
categories:
- 编程
---

## 引子
前不久在CppCon上看到一个Talk：[constexpr All the things](https://www.youtube.com/watch?v=PJwd4JLYJJY)，这个演讲主题令我非常震撼，在编译期解析json字符串，进而提出了编译期构造正则表达式（编译期构建FSM），现场掌声一片，而背后依靠的是C++强大的constexpr特性，从而大大提高了编译期计算威力。

早在C++11的时候就有constexpr特性，那时候约束比较多，只能有一条return语句，能做的事情只有简单的递归实现一些数学、hash函数；而到了C++14的时候这个约束放开了，允许像普通函数那样，进而社区产生了一系列constexpr库；而在C++17，更加泛化了constexpr，允许`if constexpr`来代替元编程的SFINAE手法，STL库的一些算法支持constexpr，甚至连lambda都默认是constexpr的了；到C++20，更加难以想象，居然支持了constexpr new，STL的vector都是constexpr的了，若用constexpr allocator和constexpr destructor，那么就能统一所有constexpr容器了。

借助C++的constexpr + lambda能力，可以轻而易举的构造Parser Combinator，实现一个Parser也没那么繁杂了，对用户定义的字符串（User defined literal）释放了巨大的潜力，这也是本文的重点。

## 什么是Parser
Parser是一个解析器函数，输入一个字符串，输出解析后的类型值集合，函数签名如下：
```haskell
Parser a :: String -> [(a, String)]
```

简单起见，这里我们考虑只输出零或一个类型值结果，而不是集合，那么签名如下：
```haskell
Parser a :: String -> Maybe (a, String)
```

举个例子，一个数字Parser，解析输入字符串`"123456"`，输出结果为`Just (1, "23456")`，即得到了数字1和剩余字符串`"23456"`，从而可以供下一个Parser使用；若解析失败，输出`None`。

对应C++的函数签名，如下：
```cpp
// Parser a :: String -> Maybe (a, String)
using ParserInput = std::string_view;
template <typename T>
using ParserResult = std::optional<std::pair<T, ParserInput>>;
template <typename T>
using Parser = auto(*)(ParserInput) -> ParserResult<T>;
```

这就是Parser的定义了。

根据定义可以实现几个最基本的Parser，例如匹配给定的字符：
```cpp
constexpr auto makeCharParser(char c) {
    // CharParser :: Parser Char
    return [=](ParserInput s) -> ParserResult<char> {
        if (s.empty() || c != s[0]) return std::nullopt;
        return std::make_pair(s[0], ParserInput(s.begin() + 1, s.size() - 1));
    };
};
```

`makeCharParser`相当于一个工厂，给定字符`c`，创建匹配`c`的Parser。

匹配给定集合中的字符：
```cpp
constexpr auto oneOf(std::string_view chars) {
    // OneOf :: Parser Char
    return [=](ParserInput s) -> ParserResult<char> {
        if (s.empty() || chars.find(s[0]) == std::string::npos) return std::nullopt;
        return std::make_pair(s[0], ParserInput(s.begin() + 1, s.size() - 1));
    };
}
```
## 什么是Parser Combinator
Parser是可组合的最小单元，Parser与Parser之间可以组合成任意复杂的Parser，而Parser Combinator就是一个高阶函数，输入一系列Parser，输出复合后的新Parser。

根据定义，可以实现一个Combinator组合两个Parser，同时根据两个Parser的结果计算出新的结果，从而得到新的Parser：
```cpp
// combine :: Parser a -> Parser b -> (a -> b -> c) -> Parser c
template<typename P1, typename P2, typename F,
    typename R = std::invoke_result_t<F, Parser_t<P1>, Parser_t<P2>>>
constexpr auto combine(P1&& p1, P2&& p2, F&& f) {
    return [=](ParserInput s) -> ParserResult<R> {
        auto r1 = p1(s);
        if (!r1) return std::nullopt;
        auto r2 = p2(r1->second);
        if (!r2) return std::nullopt;
        return std::make_pair(f(r1->first, r2->first), r2->second);
    };
};
```

由于C++支持操作符重载，那么可以重载一个二元操作符来组合两个Parser，比如从两个Parser里取出其中一个Parser的结果产生新的Parser：

取左边Parser的结果：
```cpp
// operator> :: Parser a -> Parser b -> Parser a
template<typename P1, typename P2>
constexpr auto operator>(P1&& p1, P2&& p2) {
    return combine(std::forward<P1>(p1),
                   std::forward<P2>(p2),
                   [](auto&& l, auto) { return l; });
};
```

取右边Parser的结果：
```cpp
// operator< :: Parser a -> Parser b -> Parser b
template<typename P1, typename P2>
constexpr auto operator<(P1&& p1, P2&& p2) {
    return combine(std::forward<P1>(p1),
                   std::forward<P2>(p2),
                   [](auto, auto&& r) { return r; });
};
```

有时候需要对同一个Parser进行多次匹配，类似正则表达式的`*/+`操作，这个操作可以看做是`fold`，执行多次Parser直到匹配失败，每次结果传递给一个函数运算：
```cpp
// foldL :: Parser a -> b -> (b -> a -> b) -> ParserInput -> ParserResult b
template<typename P, typename R, typename F>
constexpr auto foldL(P&& p, R acc, F&& f, ParserInput in)
    -> ParserResult<R> {
    while (true) {
        auto r = p(in);
        if (!r) return std::make_pair(acc, in);
        acc = f(acc, r->first);
        in = r->second;
    }
};
```

有了`fold`函数，那么可以很容易实现函数来匹配任意多次`many`，匹配至少一次`atLeast`：
```cpp
// many :: Parser a -> Parser monostate
template<typename P>
constexpr auto many(P&& p) {
    return [p=std::forward<P>(p)](ParserInput s)
        -> ParserResult<std::monostate> {
        return detail::FoldL(p, std::monostate{},
            [](auto acc, auto) { return acc; }, s);
    };
};
```

```cpp
// atLeast :: Parser a -> b -> (b -> a -> b) -> Parser b
template<typename P, typename R, typename F>
constexpr auto atLeast(P&& p, R&& init, F&& f) {
    static_assert(std::is_same_v<std::invoke_result_t<F, R, Parser_t<P>>, R>,
            "type mismatch!");
    return [p=std::forward<P>(p),
           f=std::forward<F>(f),
           init=std::forward<R>(init)](ParserInput s) -> ParserResult<R> {
        auto r = p(s);
        if (!r) return std::nullopt;
        return detail::foldL(p, f(init, r->first), f, r->second);
    };
};
```

还有种操作是匹配零到一次，类似于正则表达式的`?`操作，这里我定义为`option`操作：
```cpp
// option :: Parser a -> a -> Parser a
template<typename P, typename R = Parser_t<P>>
constexpr auto option(P&& p, R&& defaultV) {
    return [=](ParserInput s) -> ParserResult<R> {
        auto r = p(s);
        if (! r) return make_pair(defaultV, s);
        return r;
    };
};
```

有了以上基本操作，接下来看看如何运用。
## 实战
### 解析数值
项目中模板元编程比较多，而C++17之前模板Dependent type（非类型参数）不支持double，得C++20才支持double，临时方案就是用`template<char... C> struct NumWrapper {};`和User defined literal来模拟double的类型，而需要获取其值的时候，就需要解析字符串了，这些工作应该在编译期确定。

首先是匹配符号`+/-`，若没有符号，则认为是`+`：
```cpp
constexpr auto sign = Option(OneOf("+-"), '+');
```

其次是整数部分，也可能没有，若没有，则认为是0：
```cpp
constexpr auto number = AtLeast(OneOf("1234567890"), 0l, [](long acc, char c) -> long {
    return acc * 10 + (c - '0');
});
constexpr auto integer = Option(number, 0l);
```

然后是小数点`.`，若没有小数点，为了不丢失精度，则返回一个`long`值。
```cpp
constexpr auto point = MakeCharParser('.');
// integer
if (! (sign < integer < point)(in)) {
    return Combine(sign, integer, [](char sign, long number) -> R {
        return sign == '+' ? number : -number;
    })(in);
}
```

若有小数点，认为是浮点数，返回其`double`值。
```cpp
// floating
constexpr auto decimal = point < Option(number, 0l);
constexpr auto value = Combine(integer, decimal,
    [](long integer, long decimal) -> double {
    double d = 0.0;
    while (decimal) {
        d = (d + (decimal % 10)) * 0.1;
        decimal /= 10;
    }
    return integer + d;
});
return Combine(sign, value,
    [](char sign, double d) -> R { return sign == '+' ? d : -d; })(in);
```

由于该Parser可能返回`long`或者`double`类型，所以可以统一成和类型`std::variant`：
```cpp
constexpr auto ParseNum() {
    using R = std::variant<double, long>;
    return [](ParserInput in) -> ParserResult<R> {
        // ...
    };
}
```

最后我们的`NumWrapper`实现如下，从而可以混入模板类型体系：
```cpp
template<char... Cs>
constexpr std::array<char, sizeof...(Cs)> ToArr = {Cs...};
template<char ...Cs>
class NumberWrapper {
public:
    constexpr static auto numStr = ToArr<Cs...>;
    constexpr static auto res =
        ParseNum()(std::string_view(numStr.begin(), numStr.size()));
    static_assert(res.has_value() && res->second.empty(), "parse failed!");
public:
    constexpr static auto value =
        std::get<res->first.index()>(res->first); // long or double
}
```

如果仅仅是用于解析数字，那也杀鸡用牛刀了，因为在`Parser Combinator`之前的版本，我就是在一个普通的`constexpr`函数中完成解析的，代码很无趣，但现在我可能想回退代码了。
### Json解析导读
这次的CppCon主题是编译期解析`json`字符串，当然直接用`string_view`承载字符串即可。然后构造一些constexpr容器，例如固定长度的constexpr vector，由于是17年的talk了，在还不支持constexpr new的情况下，只能这么做。有了constexpr vector，进而可以构造map容器，也是很简单的pair vector集合。

进而提出Parser Combinator，解析字符串，`fmap`到json数据结构中。

最初实现的时候，json数据结构也是一个大的`template<size_t Depth> struct Json_Value;`模板承载，导致只能指定最大递归层数，那就不够实用了。然后talker想了个很巧妙的办法去掉层数约束，就是先递归`sizes()`扫描一遍，计算出所有值个数，这样就能确定需要多少个`Value`容器来存储，其次计算出字符串长度，由于`UTF8`、转义字符串的影响，最终要解析的长度其实是可能小于输入长度的。有了确定空间后，进行第二遍递归`value_recur<NumObjects, StringSize>::value_parser()`扫描，每次解析完整值时候填一下`Value`数据结构。而由于数组和对象类似，可能嵌套，这时候进行第三遍递归`extent_recur<>::value_parser()`扫描，相当于做一次宽度优先搜索，确定最外层的元素个数，从而依次解析填值。

## 后记
关于编译速度，很多人觉得可能会很慢，实际上不会，talker用他5年前的笔记本电脑编译解析那么复杂的json，才花了5秒时间，而在我的MBP编译，也是秒级。何况牺牲编译速度换来零成本运行时，是值得的。

这也是C++多年来的进步，据说曾经的boost.sprit库解析一个简单的url字符串都需要编译1分多钟，可想而知这个进步速度多大了。
