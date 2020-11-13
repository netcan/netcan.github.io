---
title: 详解函数式编程之Monad
toc: true
original: true
date: 2020-09-30 21:06:07
tags:
- 函数式编程
categories:
- 学习笔记
---

## 前言
最近终于搞清楚了Monad的本质，趁热记录下来，相信大家或多或少在编程语言中见过并用过，只不过不知道那是Monad罢了，也为了方便大家理解Monad，后面我会用各种主流语言中具有代表性的Monad作为例子，如果对理论不感兴趣可以直接跳到后面，寻找你熟悉语言的例子进行理解后，再回头看看理论。

## 何为Monad
Monad概念最初来自于范畴论，后来引入进函数式编程语言，由函数式编程发扬光大，渐渐渗透到其他主流语言中。

FP中具有代表性的语言是Haskell，其特点是函数是第一公民，延迟计算、引用透明^[函数无副作用，输出完全由输入决定]。正因为是延迟计算，所以对表达式求值时顺序是不确定的，然而在需要确定顺序的场合，比如IO，在屏幕中打印字符串要求用户输入，然后等待用户输入，若顺序不确定，很有可能颠倒顺序，即先等待用户输入，然后打印字符串要求用户输入[^2]。而Monad能做到确定顺序即确定控制流，像**过程式**语言一条条语句按顺序执行那样；其次是Monad能屏蔽副作用，例如在IO Monad中，所有对IO操作都被屏蔽在Monad上下文中，从而确定了一个边界，也没违背FP的无副作用性质。

关于确定控制流，其实不只有Monad这条路，还有经典的Continuation passing style(CPS)风格，经过CPS变换可以实现函数在任意点挂起，后续恢复执行，没错，这也是协程的特性，C++20协程的`await_suspend`接口传递的就是Continuation供后续恢复上下文；然而CPS风格代码可读性过差，回调嵌套回调，层数一多智商不够用了。好在Monad提供了一种友好的方式，毕竟程序猿更加能接受的是符合直觉的串行代码，而不是回调嵌套代码。

## Monad定义
这部分是关于Monad的理论，Monad是一个类型构造子（下面简称`M`），定义了两个操作：

1. `unit :: t -> M t`，将类型T **包装**成Monad类型，也被常常称为`return/pure/lift`操作
2. `bind :: M a -> (a -> M b) -> M b`，bind组合子，输入一个`M a`和一个函数然后返回`M b`，输入的函数可以拿到**被包装**的类型`a`，进行变换返回`M b`。这个操作定义了二元操作符，在haskell里用`>>=`操作符表示。

这两个操作可以这样理解，定义个Monad类型用于**包裹**另一个类型`T`，同时定义一个二元操作，左边对象是个 `Monad<T>`，右边对象是个函数接受被包裹的 `T` 值并返回一个 `Monad<U>`对象，这样就能不断通过这个二元操作串起来。

在haskell中定义[^3]就是：
```haskell
class Monad m where
  return :: a -> m a
  (>>=) :: forall a b . m a -> (a -> m b) -> m b
```

只要满足这两个定义，就可以看做是Monad，其有些比较有趣的性质：

1. `join :: M (M a) -> M a`，join操作可以看做是去掉一层Monad，这样性质比较重要，可以**去掉嵌套**。
2. `unit`是`>>=`运算的单位元
    - 左单位元：`unit t >>= f(t) ↔ f(t)`
    - 右单位元：`m a >>= unit ↔ m a`
3. `>>=`操作满足结合律：`m a >>= \x -> (f x >>= g) ↔ (m a >>= f) >>= g`

## 主流语言的常见Monad
### Maybe
Maybe其实是一种非常简单的Monad，它的概念是表达一个值可有可无，当值被封装到Maybe的世界里，它其实有两种可能：要么拥有值即Just，要么没有即Nothing。根据Monad的性质，Maybe应该有如下操作：

- 将值包装成Maybe类型
- 二元操作，将Maybe类型与操作它的函数串起来，产生新的Maybe类型

前者用于构造Maybe Monad，后者用于串联一系列Maybe类型，避免写一堆if/match、取值语句，况且从Maybe取出其不存在的值是有可能抛异常的。

既然Maybe可以包装一个值，那么该值可能也是Maybe的，即`Maybe<Maybe<T>>`，这种情况下实际取值情况有3种：最外层的`Just<Maybe<T>>`和`Nothing`，而`Just<Maybe<T>>`取值又有两种：`Just<Just<T>>`和`Just<Nothing>`，逻辑上讲其实应该只有两种取值的：`Just<T>`和`Nothing`。那么需要一层层判断取值么？这种情况就得写两个嵌套if/match，若类型为`Maybe<Maybe<Maybe<T>>>`，难道需要写嵌套3个if/match么？还记得Monad有个有趣的性质是join操作么，可以通过不断join去除外层的Monad，最终flatten成一个Monad即：`join Maybe<Maybe<T>> -> Maybe<T>`，从而减少if/match嵌套判断。

很多语言都提供了这种概念，比如Haskell的Maybe，Rust的Option，C++的std::optional。

#### Rust
Rust的Option其实做的相当完备，我们看看其定义：
```rust
enum Option<T> {
    None,
    Some(T),
}
```

和关键的Monad操作：

- `None/Some<T>`提供了将值包装成`Option<T>`的可能，即`unit`操作
- `fn and_then<U, F>(self, f: F) -> Option<U>`实现了二元操作，即`bind`操作

根据例子看看这两个操作是怎么结合的：
```rust
fn sq(x: u32) -> Option<u32> { Some(x * x) }
fn nope(_: u32) -> Option<u32> { None }

assert_eq!(Some(2).and_then(sq).and_then(sq), Some(16)); // 串联操作
assert_eq!(Some(2).and_then(sq).and_then(nope), None);
assert_eq!(Some(2).and_then(nope).and_then(sq), None); // 串联操作，避免判断None
assert_eq!(None.and_then(sq).and_then(sq), None);
```

可惜Rust不支持操作符重载，只能显式的`and_then`。顺便一提，Rust的Option也提供了join操作，即 `fn flatten(self) -> Option<T>`函数。

#### Haskell
Haskell的Maybe定义如下：
```haskell
data Maybe a = Just a | Nothing
```

而Maybe又是Monad，定义了unit和bind操作：
```haskell
instance Monad Maybe where
    return  a           = Just a
    (Just x) >>= f      = f x
    Nothing  >>= _      = Nothing
```

看看例子：
```haskell
sq x   = Just (x * x)
nope _ = Nothing
*Main> Just 2 >>= sq >>= sq
Just 16

*Main> Just 2 >>= sq >>= nope
Nothing

*Main> Just 2 >>= nope >>= sq
Nothing

*Main> Nothing >>= sq >>= sq
Nothing
```

和Rust那个例子一模一样，除了用`>>=`代替`and_then`。Haskell还提供了一种语法糖do，看起来更加直观，就和命令式语言写法差不多：
```haskell
do
    v1 <- Just 2
    v2 <- sq v1
    v3 <- sq v2
    return v3 -- Just 16
```

#### C++
C++17提供的`std::optional`其实不够完备，得自己封装一下，不适合拿出来当Monad讲，所以我用C++实现了一个[Maybe.cpp](https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/Maybe.cpp)用于展示Monad的概念。

Maybe定义如下：
```cpp
template<typename T>
struct Just {
    T value;
    operator const T&() const { return value; }
    Just(const T& value): value(value) {}
};
struct Nothing {};

template<typename T>
struct Maybe {
    std::variant<Just<T>, Nothing> value;
    Maybe(const T& value): value(value) {}
    Maybe(const Nothing&): value(Nothing{}) {}
};
```

构造函数即`unit`操作，再来定义一个`bind`操作：
```cpp
template<typename T>
auto just(T x) -> Maybe<T> { return x; }

struct AnyType {
    template<typename T> operator T();
    friend std::ostream& operator<<
    (std::ostream& os, const AnyType&) { return os; };
};
auto nothing() -> Maybe<AnyType> { return Nothing{}; }

template<typename T, typename F>
auto operator>>=(const Maybe<T>& ma, F&& f) {
    using R = std::invoke_result_t<F, T>;
    return std::visit(overloaded {
        [&](const Just<T>& v) -> R
        { return f(static_cast<T>(v)); },
        [](Nothing) -> R { return Nothing{}; }
    }, ma.value);
}
```

同样提供例子看看，可以发现和Haskell很相似。
```cpp
show((just(3) >>=
    [](int v) -> Maybe<double> {
        return {v * 1.5};
    }) >>= [](double v) -> Maybe<double> {
        return {v * 1.5};
    }); // Just 6.75

show((nothing() >>=
    [](int v) -> Maybe<double> {
        return {v * 1.5};
    }) >>= [](double v) -> Maybe<double> {
        return {v * 1.5};
    }); // Nothing
```

### IO Monad
在FP中一切都是纯函数，那么对于IO这种拥有副作用的功能，怎么融入到FP的世界中去呢？有了IO Monad包裹了对IO的操作动作，副作用被约束在Monad中，同时也提供了与外界环境交互的边界。

Haskell提供了最基本的两个IO操作：

- `getLine :: IO String`，从stdin读取一行，unit操作
- `print :: Show a => a -> IO ()`，输出一行到stdout

观察`getLine`发现函数返回类型是个`IO String`，也就是读到的字符串被包裹在了IO Monad中，用户没法直接操作Monad里面的值，那么若要对读到的字符串进行操作，就得通过bind操作，将动作隔离在Monad的世界里，同时该操作返回类型也必须是个IO Monad，这样就不会将副作用带出来了。

举个例子，要求用户输入一个数，然后将该数乘以二，最后打印在屏幕上，代码这样写：
```cpp
putStrLn "input a number"
>>= \_ -> getLine
>>= \str -> print $ (read str :: Int) * 2
```

这个表达式的类型是`IO ()`，对输入的数乘以二操作也被隔离在IO Monad的世界里了，普通函数没法直接拿到IO世界里的值，必须得将函数lift到这个世界里与其共舞。同样的由于Monad确定了控制流，保证了按代码顺序执行，先getLine后print。

### Writer Monad
有时候我们需要对每一步操作进行日志记录，简单起见，但是不想每次操作后进行日志拼接工作，若抽象成Monad，可以让bind帮我们完成这个操作，这节我打算用Javascript来快速实现个原型，方便熟悉Javascript的同学理解。

Writer定义为一个序对，第一个存放函数执行结果，第二个存日志：`[value, string]`。

unit操作定义如下：
```javascript
const unit = value => [value, ""];
```

bind操作对value执行函数f，接着拼接日志，然后返回一个Writer。
```javascript
const bind = (writer, f) => {
    const [value, log] = writer
    const [result, updates] = f(value)
    return [result, log + updates]
};
```

由于bind操作输入一个Writer最终输出的也是Writer，那么可以考虑写个函数将一系列操作串起来：
```javascript
const pipelog = (writer, ...fs) =>
    fs.reduce(bind, writer);
```

最后看看例子：
```javascript
const doubled = x => [x*2, `${x} was doubled.\n`]
const halved = x => [x/2, `${x} was halved.\n`]
pipelog(unit(2), doubled, doubled, doubled, halved)
// [8, "2 was doubled.↵4 was doubled.↵8 was doubled.↵16 was halved.↵"]
```

### Parser Combinator
再来看看一个比较复杂的例子，Parser也是一个Monad，经过bind操作可以将各种类型的Parser串起来，解析字符串输出不同的结构。而Parser定义并不是像前面一样是个类型容器结构，而是一个函数：
```haskell
Parser a :: String -> [(a, String)]
```

Parser包裹了要得到的类型，通过执行Parser函数，解析输入一个字符串，输出所需要的类型，具体细节可以看我之前写的一篇文章：[C++元编程之Parser Combinator](/2020/09/16/C-元编程之Parser-Combinator/)，其中的`makeCharParser/oneOf`就是不同种类的unit操作，同时定义了个类似bind操作的`combine`和二元操作符`operator>/operator<`。

### Promise/Future
最后一个例子，也是主流的概念，相当重要，几乎是写异步编程必须要掌握的概念，通过Monad操作，以同步（串行）方式写异步代码！

Promise/Future[^4]是上个世纪七十年代就提出来的概念，顾名思义，就是我给你的承诺Promise，将来Future某个时间点履行承诺。通过Promise可以得到一个Future，而Future包裹了未来的值，直到Promise将值设定，Future最终可以取出值，在这之前是拿不到值的（毕竟都没有，怎么拿）。

那么Future做成一个Monad会怎么样？unit操作就是通过Promise得到`Future<T>`，而bind二元操作就是输入一个`Future<T>`和动作，该动作可以拿到未来的值`T`，进行处理后返回一个新`Future`，这样就能将各个Future串起来。

既然Future是对未来的预期，当下可能没有值，自然拿不到值，但是我们可以通过bind操作将处理值的动作串起来带入Future的世界里，将来某个时间点执行这个动作，这也是以同步方式写异步代码！FP的Monad概念完美解决了传统语言异步编程需要挂各种回调的难处。

若像Haskell的do语法糖那样引入await关键字，异步代码看起来就更加串行了。

#### Javascript
最初我是通过Javascript了解异步编程方式，回过头来一看，其Promise也是一个Monad，然而Future概念也收到Promise里去了。

对于unit操作，其实是Promise的构造函数，传递一个动作，得到一个Promise，表示将来会执行这个动作：
```javascript
const myPromise = new Promise((resolve, reject) => {
    // condition
});
```

而bind二元操作，对应于Promise的then接口，传递一个动作执行，返回一个Promise，从而能将各个Promise串起来。

看一个例子感受下用Monad前后怎么将异步代码同步化的：
```javascript
firstRequest(response =>
    secondRequest(response, nextResponse =>
        thirdRequest(nextResponse, finalResponse =>
            console.log('Final response: ' + finalResponse);
            , failureCallback),
        failureCallback),
    failureCallback);
```

使用Monad后，大大简化回调接口：
```javascript
firstRequest()
    .then(response      => secondRequest(response))
    .then(nextResponse  => thirdRequest(nextResponse))
    .then(finalResponse => console.log('Final response: ' + finalResponse))
    .catch(failureCallback);
```

最后再引入await语法糖，看上去更加串行了，这也是协程的提供的手段：
```javascript
try {
    response      = await firstRequest();
    nextResponse  = await secondRequest(response);
    finalResponse = await thirdRequest(nextResponse):
    console.log('Final response: ' + finalResponse);
} catch (e) {
    failureCallback(e);
}
```

## 尾声
终于写到最后了，其实我们用了很久的Monad，不同的Monad背后高度一致性，将控制流封装起来，抽象掉对上下文环境的处理，从而提高代码的可读性。应用多了回头理解抽象的Monad概念，其实也不是那么难了，之后理解这类概念就更加得心应手了，这也是抽象之美。

[^2]: [http://blog.sigfpe.com/2006/08/you-could-have-invented-monads-and.html](http://blog.sigfpe.com/2006/08/you-could-have-invented-monads-and.html)
[^3]: [https://hackage.haskell.org/package/base-4.14.0.0/docs/src/GHC.Base.html#%3E%3E%3D](https://hackage.haskell.org/package/base-4.14.0.0/docs/src/GHC.Base.html#%3E%3E%3D)
[^4]: [https://en.wikipedia.org/wiki/Futures_and_promises](https://en.wikipedia.org/wiki/Futures_and_promises)
