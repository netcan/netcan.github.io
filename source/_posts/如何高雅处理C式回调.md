---
title: 如何高雅地处理C式回调
toc: true
original: true
date: 2025-01-04 21:24:25
tags:
- C/C++
- 元编程
categories:
- 编程
---

在C类接口中，我们经常能看到这种定义：

```cpp
// Callback signature for data getter.
typedef ImPlotPoint (*ImPlotGetter)(int idx, void* user_data);
void PlotLineG(const char* label_id, ImPlotGetter getter_func, void* data, int count, ImPlotLineFlags flags);
```

虽然这是个C++库，但不得不采用比较恶心的C式回调，导致无法传递lambda，用户必须得手动传递回调所需的context，这也就是典型的`user_data`：

```cpp
static ImPlotPoint DataGetter(int idx, void* user_data) {
    auto& self = *reinterpret_cast<MyPlot*>(user_data); // <- 变化点1
    size_t i = idx + std::lround(self.mXLimit.Min);
    if (i >= self.f.size()) { // <- 变化点2
        return ImPlotPoint(i, std::nan(""));
    }
    return ImPlotPoint(i, self.f[i]);
}

ImPlot::PlotLineG("F", DataGetter, &this, this->f.size());
```

问题来了：如果有不同的类，每个类的处理逻辑几乎一样：首先判断是否处于范围内，然后返回有效数据，那么就得写很多这类`DataGetter`函数，并且对于同一类的`DataGetter`函数，又需要处理不同的数据字段，有没有什么很好的办法能够复用代码呢？

假如是C++接口，那么很容易通过捕获this指针的方式构造各种各样的lambda传递。针对这种C式回调，也有很优雅的处理方式：

```cpp
ImPlot::PlotLineG("F", DataGetterBuilder<[](MyPlot& self, int idx)
                       { return ImPlotPoint(idx, self.ma[idx]); }>,
                  this, mXLimit.Size());
```

可以看到上面这种方式，能够将变化点（类、数据）剥离出来，不变点（校验）封装到`DataGetterBuilder`中，接下来重点就是如何实现`DataGetterBuilder`，签名如下：

```cpp
template<auto Func>
constexpr ImPlotGetter DataGetterBuilder = /* implementation */
```

`DataGetterBuilder`是一个编译期常量，它接受一个（类成员）函数指针，并生成一个`ImPlotGetter`签名的函数，从而能够被传递，这也是高阶函数：返回函数的函数。

在使用的时候`Func`函数指针需要为无捕获的lambda函数，这样它才能保留函数指针的性质，一旦携带了捕获，那么就无法转成函数指针，自然也无法被C式回调所接受：

```cpp
template<auto Func>
constexpr ImPlotGetter DataGetterBuilder = [](int idx, void* user_data) -> ImPlotPoint{
    using Self = std::decay_t<FirstArgT<decltype(&decltype(Func)::operator())>>;
    auto& self = *static_cast<Self*>(user_data);
    size_t i = idx + std::lround(self.mXLimit.Min);
    if (i >= self.size()) {
        return ImPlotPoint(i, std::nan(""));
    }
    return Func(self, i);
};
```

第二个有意思的是如何获得`Func`函数的第一个类型，从而能够完成`void*`到自定义类型的转换呢？C++17之前可以通过`std::function<F>::first_argument_type`来拿到，但是在C++20中被废弃掉了，这里不得不重新造轮子：`FirstArgT`：

```cpp
template<typename F>
struct FirstArgType;

template <typename R, typename C, typename A1, typename... Args>
struct FirstArgType<R (C::*)(A1, Args...) const> : std::type_identity<A1> {};

template <typename R, typename C, typename A1, typename... Args>
struct FirstArgType<R (C::*)(A1, Args...)> : std::type_identity<A1> {};

template <typename R, typename A1, typename... Args>
struct FirstArgType<R (*)(A1, Args...)> : std::type_identity<A1> {};

template<typename F>
using FirstArgT = FirstArgType<F>::type;
```

通过`decltype(Func)`可以得到具体的lambda类型，其成员`decltype(Func)::operator()`就是最终用户传递的函数签名，通过它来拿到第一个函数参数类型为何，这样就能做到`void*`对自定义类型的转换。


通过以上技巧，就能够优雅解决传递C式回调传递context问题，以及`void*`自动地转换成所需类型。
