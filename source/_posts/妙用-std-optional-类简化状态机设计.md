---
title: '妙用 std::optional 类简化状态机设计'
toc: true
original: true
date: 2022-05-16 14:49:10
tags:
- C/C++
categories:
- 编程
---

一些类可能存在和生命周期相关的状态，例如该类中常常包含着`Start`和`Stop`等接口，通常而言，对象被构造动作相当于做初始化，而`Start`方法才真正干活，例如创建线程，然后`Stop`方法去停掉线程，那么对于析构函数而言，除了本身需要销毁的那部分动作外，还需要检查`Start`的状态，以便判断是否需要`Stop`。

一旦引入`Start`和`Stop`等接口，意味着该类的作者需要追踪这个类的状态，并且这个类的每个接口可能都需要判断这个状态，总之，他需要考虑的因素也多了些：

- 如果用户重复`Start`怎么办？
- 如果用户仅仅`Stop`怎么办？
- 某个接口调用的时候，是否需要保证`Start`状态？如果没有怎么处理？
- 析构行为需要考虑`Start`状态

如果将`Start`的语义收编到构造函数，`Stop`语义收编到析构函数，那么因为该状态的消除意味着复杂度也降了很多，代码因此也就很简洁。如果用户仍然需要延迟`Start`的语义，那么可以交由类`std::optional`组合而成，例如在需要`Start`的地方使用`emplace`接口构造出该对象，在需要`Stop`的地方使用`reset`接口销毁对象：

```cpp
class UserObj {
    // ...
private:
    // 使用optional表达额外的Start/Stop语义
    std::optional<Thread> m_thread;
};
```

`std::optional<Thread>`类由C++17标准引入，它相当于组合了状态类与标记字段，并且无需动态分配内存，可以理解大小为`sizeof(Thread) + sizeof(bool)`，这要比使用`unique_ptr<Thread>`的方式要高效的多。
