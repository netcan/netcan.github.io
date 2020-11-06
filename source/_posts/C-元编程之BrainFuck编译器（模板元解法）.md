---
title: C++元编程之BrainFuck编译器（模板元解法）
toc: true
original: true
date: 2020-11-04 21:56:49
tags:
- C/C++
- 元编程
categories:
- 编程
---

## 引子
昨天在逼乎看到一个有意思的问题：[怎么使C++用最复杂的方法打hello world？](https://www.zhihu.com/question/334884938/answer/1557544809)，既然要用最有格调、最复杂的方式，那就不要考虑直接打印hello world这种方式了，而是用元编程这种手段，**编译期**生成hello world字符串，运行时打印。

编译期怎么生成字符串呢？这里我想到了BrainFuck语言[^1]，它是图灵完备的，同时其hello world程序又足够唬人（`;`为注释）：

```brainfuck
std::cout << R"(
>++++++++[<+++++++++>-]<.                 ; H
>>++++++++++[<++++++++++>-]<+.            ; e
>>+++++++++[<++++++++++++>-]<.            ; l
>>+++++++++[<++++++++++++>-]<.            ; l
>>++++++++++[<+++++++++++>-]<+.           ; o
>>++++[<++++++++>-]<.                     ;
>>+++++++++++[<++++++++>-]<-.             ; W
>>++++++++++[<+++++++++++>-]<+.           ; o
>>++++++++++[<++++++++++++>-]<------.     ; r
>>+++++++++[<++++++++++++>-]<.            ; l
>>++++++++++[<++++++++++>-]<.             ; d
>>++++++[<++++++>-]<---.                  ; !
)"_brain_fuck << std::endl;
```

若元编程实现一个BrainFuck编译器，在**编译时**解析执行上述代码，并生成结果字符串（hello world），那多有意思🤔

上述代码执行过程可以看看这个链接感受下 [brainfuck-visualizer](https://fatiherikli.github.io/brainfuck-visualizer/#PisrKysrKysrWzwrKysrKysrKys+LV08LiAgICAgICAgICAgICAgICAgOyBICj4+KysrKysrKysrK1s8KysrKysrKysrKz4tXTwrLiAgICAgICAgICAgIDsgZQo+PisrKysrKysrK1s8KysrKysrKysrKysrPi1dPC4gICAgICAgICAgICA7IGwKPj4rKysrKysrKytbPCsrKysrKysrKysrKz4tXTwuICAgICAgICAgICAgOyBsCj4+KysrKysrKysrK1s8KysrKysrKysrKys+LV08Ky4gICAgICAgICAgIDsgbwo+PisrKytbPCsrKysrKysrPi1dPC4gICAgICAgICAgICAgICAgICAgICA7Cj4+KysrKysrKysrKytbPCsrKysrKysrPi1dPC0uICAgICAgICAgICAgIDsgVwo+PisrKysrKysrKytbPCsrKysrKysrKysrPi1dPCsuICAgICAgICAgICA7IG8KPj4rKysrKysrKysrWzwrKysrKysrKysrKys+LV08LS0tLS0tLiAgICAgOyByCj4+KysrKysrKysrWzwrKysrKysrKysrKys+LV08LiAgICAgICAgICAgIDsgbAo+PisrKysrKysrKytbPCsrKysrKysrKys+LV08LiAgICAgICAgICAgICA7IGQKPj4rKysrKytbPCsrKysrKz4tXTwtLS0uICAgICAgICAgICAgICAgICAgOyAhCg==)

## BrainFuck编译器设计
如果点赞数足够多，我会考虑换纯constexpr方式实现，这篇采用传统的模板元方式。

我们需要C++在**编译期**解析执行BrainFuck代码，所以无法进行IO，这里忽略掉`,/.`运算符，执行完BrainFuck代码后，需要将单元结果保存起来。

我们需要抽象出如下几个概念：

- `Cell`单元格，用于存放一个char值
- `Machine`，BrainFuck图灵机，包含一组`Cell`，一个指针指明当前操作第几个`Cell`

由于在编译期计算操作的都是常量，模板元编程采用的是函数式编程思想，那就需要用递归解决问题。

而循环有如下几种情况，一一分析

- `[`操作符
    - 若当前单元格值不为零，进入循环
    - 否则跳过该循环，执行`]`后的代码
- `]`操作符
    - 若当前单元格值不为零，进入下一轮循环，从之前`[`的代码开始执行
    - 否则跳过该循环，执行`]`后的代码

而递归解析到操作符`[`时，有两条路可走，要么进入循环，要么跳到`]`，由于当前递归解析无法知道下一个`]`位置，那么可以通过标记位`skip`来判断是否继续执行，若`skip`为真则不执行代码，直到`]`复位`skip`标记位为止（即跳过本次循环）。

同样地，递归解析到`]`时，也有两条路可走，要么进入下一轮循环，跳到前面的`[`位置，要么结束本次循环继续执行后面代码。而递归解析没有记录之前`[`的位置，那么可以通过回溯到`[`位置进行决策要不要进入下一轮循环，同样思路，可以通过标记位`InLoop`判断是否需要进入循环。

完整实现可看：[Brainfuck.cpp](https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/brain_fuck/BrainFuckTemplateMeta.cpp)，运行结果：[https://godbolt.org/z/GTKxhc](https://godbolt.org/z/GTKxhc)

从汇编结果来看，仅仅生成12行汇编代码，相当紧凑简洁了：

```nasm
main:
        subq    $8, %rsp
        movl    $MachineTrait::ToStr<11ul, false, std::integral_constant<char, (char)72>, std::integral_constant<char, (char)101>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)32>, std::integral_constant<char, (char)87>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)114>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)100>, std::integral_constant<char, (char)33>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0> >(Machine<11ul, false, std::integral_constant<char, (char)72>, std::integral_constant<char, (char)101>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)32>, std::integral_constant<char, (char)87>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)114>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)100>, std::integral_constant<char, (char)33>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0> >)::str, %edi
        call    puts
        xorl    %eax, %eax
        addq    $8, %rsp
        ret
MachineTrait::ToStr<11ul, false, std::integral_constant<char, (char)72>, std::integral_constant<char, (char)101>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)32>, std::integral_constant<char, (char)87>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)114>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)100>, std::integral_constant<char, (char)33>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0> >(Machine<11ul, false, std::integral_constant<char, (char)72>, std::integral_constant<char, (char)101>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)32>, std::integral_constant<char, (char)87>, std::integral_constant<char, (char)111>, std::integral_constant<char, (char)114>, std::integral_constant<char, (char)108>, std::integral_constant<char, (char)100>, std::integral_constant<char, (char)33>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0>, std::integral_constant<char, (char)0> >)::str:
        .string "Hello World!"
        .string ""
        .string ""
        .string ""
```

## 实现
定义`Cell`单元格存放char值：

```cpp
template<char c>
using Cell = std::integral_constant<char, c>;
```

定义BrainFuck图灵机：
```cpp
template<size_t P = 0, bool INLOOP = false, typename ...Cells>
struct Machine {
    using type = Machine<P, INLOOP, Cells...>;
    constexpr static size_t PC = P; // 当前单元格指针
    constexpr static size_t InLoop = INLOOP; // ]是否进入下一轮循环
};
```

然后需要一些元函数对图灵机进行操作：

- 初始化图灵机：`InitMachine`
- 判断当前单元格是否为零：`IsZero`
- 对单元格`+1/-1`：`Inc/Dec`
- 左移右移单元格指针：`Left/Right`
- 置位、复位InLoop标记位：`EnableLoop/DisableLoop`
- 结果转存到字符串：`ToStr`

```cpp
namespace MachineTrait {
    template<typename T, typename U>
    struct Concat;
    template<typename T, typename U>
    using Concat_t = typename Concat<T, U>::type;
    template<size_t P, bool Q, size_t L, bool R, typename ...Ts, typename ...Us>
    struct Concat<Machine<P, Q, Ts...>, Machine<L, R, Us...>>: Machine<P, Q, Ts..., Us...> {};

    template<size_t N>
    struct InitMachine: Concat_t<Machine<0, 0, Cell<0>>, typename InitMachine<N-1>::type> {};
    template<size_t N>
    using InitMachine_t = typename InitMachine<N>::type;
    template<>
    struct InitMachine<0>: Machine<0, 0, Cell<0>> {};

    template<typename MACHINE>
    struct IsZero;
    template<typename MACHINE>
    using IsZero_t = typename IsZero<MACHINE>::type;
    template<size_t PC, bool INLOOP, typename C, typename... Cells>
    struct IsZero<Machine<PC, INLOOP, C, Cells...>>: IsZero<Machine<PC - 1, INLOOP, Cells...>> {};
    template<bool INLOOP, typename C, typename... Cells>
    struct IsZero<Machine<0, INLOOP, C, Cells...>>: std::bool_constant<C::value == 0> {};

    template<typename MACHINE>
    struct Inc;
    template<typename MACHINE>
    using Inc_t = typename Inc<MACHINE>::type;
    template<size_t PC, bool INLOOP, typename C, typename... Cells>
    struct Inc<Machine<PC, INLOOP, C, Cells...>>:
        Concat_t<Machine<PC, INLOOP, C>, Inc_t<Machine<PC - 1, INLOOP, Cells...>>> {};
    template<bool INLOOP, typename C, typename... Cells>
    struct Inc<Machine<0, INLOOP, C, Cells...>>:
        Machine<0, INLOOP, Cell<C::value + 1>, Cells...> {};

    template<typename MACHINE>
    struct Dec;
    template<typename MACHINE>
    using Dec_t = typename Dec<MACHINE>::type;
    template<size_t PC, bool INLOOP, typename C, typename... Cells>
    struct Dec<Machine<PC, INLOOP, C, Cells...>>:
        Concat_t<Machine<PC, INLOOP, C>, Dec_t<Machine<PC - 1, INLOOP, Cells...>>> {};
    template<bool INLOOP, typename C, typename... Cells>
    struct Dec<Machine<0, INLOOP, C, Cells...>>:
        Machine<0, INLOOP, Cell<C::value - 1>, Cells...> {};

    template<typename MACHINE>
    struct Right;
    template<typename MACHINE>
    using Right_t = typename Right<MACHINE>::type;
    template<size_t PC, bool INLOOP, typename... Cells>
    struct Right<Machine<PC, INLOOP, Cells...>>:
        Machine<PC+1, INLOOP, Cells...> {};

    template<typename MACHINE>
    struct Left;
    template<typename MACHINE>
    using Left_t = typename Left<MACHINE>::type;
    template<size_t PC, bool INLOOP, typename... Cells>
    struct Left<Machine<PC, INLOOP, Cells...>>:
        Machine<PC-1, INLOOP, Cells...> {};

    template<typename MACHINE>
    struct EnableLoop;
    template<typename MACHINE>
    using EnableLoop_t = typename EnableLoop<MACHINE>::type;
    template<size_t PC, bool INLOOP, typename... Cells>
    struct EnableLoop<Machine<PC, INLOOP, Cells...>>:
        Machine<PC, true, Cells...> {};

    template<typename MACHINE>
    struct DisableLoop;
    template<typename MACHINE>
    using DisableLoop_t = typename DisableLoop<MACHINE>::type;
    template<size_t PC, bool INLOOP, typename... Cells>
    struct DisableLoop<Machine<PC, INLOOP, Cells...>>:
        Machine<PC, false, Cells...> {};

    template<size_t PC, bool INLOOP, typename ...Cells>
    inline const auto ToStr(Machine<PC, INLOOP, Cells...>) {
        constexpr const static char str[] = { Cells::value ...  };
        return str;
    }
};
```

接下来是解析器部分，输入图灵机和BrainFuck代码，输出结果。基本就是模式匹配各种操作符，递归解析。

```cpp
template<typename MACHINE, bool skip, char ...cs>
struct BrainFuck: MACHINE {};

template<typename MACHINE, bool skip, char ...cs>
using BrainFuck_t = typename BrainFuck<MACHINE, skip, cs...>::type;

template<typename MACHINE, char c, char ...cs>
struct BrainFuck<MACHINE, true, c, cs...>: BrainFuck_t<MACHINE, true, cs...> {};

template<typename MACHINE, bool skip, char c, char ...cs>
struct BrainFuck<MACHINE, skip, c, cs...>: BrainFuck_t<MACHINE, skip, cs...> {};

template<typename MACHINE, char ...cs>
struct BrainFuck<MACHINE, false, '+', cs...>:
    BrainFuck_t<MachineTrait::Inc_t<MACHINE>, false, cs...> {};

template<typename MACHINE, char ...cs>
struct BrainFuck<MACHINE, false, '-', cs...>:
    BrainFuck_t<MachineTrait::Dec_t<MACHINE>, false, cs...> {};

template<typename MACHINE, char ...cs>
struct BrainFuck<MACHINE, false, '<', cs...>:
    BrainFuck_t<MachineTrait::Left_t<MACHINE>, false, cs...> {};

template<typename MACHINE, char ...cs>
struct BrainFuck<MACHINE, false, '>', cs...>:
    BrainFuck_t<MachineTrait::Right_t<MACHINE>, false, cs...> {};

template<typename MACHINE, char ...cs>
struct BrainFuck<MACHINE, false, '[', cs...> {
    using EnableLoopedMachine = MachineTrait::EnableLoop_t<MACHINE>;

    template<typename IN, bool = MachineTrait::IsZero_t<IN>::value>
    struct Select;
    template<typename IN> // skip
    struct Select<IN, true>: BrainFuck_t<IN, true, cs...> {};
    template<typename IN> // loop
    struct Select<IN, false>: BrainFuck_t<IN, false, cs...> {};

    using Result = typename Select<EnableLoopedMachine>::type;

    template<typename IN, bool =
        (! MachineTrait::IsZero_t<IN>::value && EnableLoopedMachine::InLoop == IN::InLoop)>
    struct Loop;
    template<typename IN> // continue
    struct Loop<IN, true>: BrainFuck_t<IN, false, '[', cs...> {};
    template<typename IN> // skip
    struct Loop<IN, false>: IN {};

    using type = typename Loop<Result>::type;
};

template<typename MACHINE, char ...cs>
struct BrainFuck<MACHINE, false, ']', cs...> {
    using DisableLoopMachine = MachineTrait::DisableLoop_t<MACHINE>;

    template<typename IN, bool = MachineTrait::IsZero_t<IN>::value>
    struct Select;
    template<typename IN> // skip
    struct Select<IN, true>: BrainFuck_t<DisableLoopMachine, false, cs...> {};

    template<typename IN> // goback
    struct Select<IN, false>: MACHINE {};

    using type = typename Select<MACHINE>::type;
};
```

最后就是通过自定义字符串模板操作符，输入BrainFuck代码，得到解析执行后的结果：
```cpp
template<typename T, T... cs>
const auto operator ""_brain_fuck() {
    using Machine = MachineTrait::InitMachine_t<15>;
    using Result = BrainFuck_t<Machine, false, cs...>;

    return MachineTrait::ToStr(Result{});
};
```

[^1]: [https://zh.wikipedia.org/wiki/Brainfuck](https://zh.wikipedia.org/wiki/Brainfuck)
