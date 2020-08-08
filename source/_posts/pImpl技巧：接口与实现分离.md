---
title: pImpl技巧：接口与实现分离
date: 2020-03-12 18:56:40
original: true
toc: true
tags:
  - C/C++
categories:
  - 编程
---

最近在实现某个模块加强了对`pImpl`设计模式的理解，这里记录下来备忘。

`pImpl`模式用来将接口与实现分离，更改实现的话只需要重新编译cpp实现，不影响依赖其抽象的模块，也就是著名的桥接模式。

然而在实践中发现有很多类嵌套类，这种也算是`pImpl`的体现，但是做的不够完美，因为这些嵌套类直接定义在头文件中，头文件需要暴露出去，其他模块自然也会看到这些内部嵌套类的定义，影响依赖关系。比如：
```cpp
// Module.h
class Module {
public:
    ...
private:
    class InternalModule { // Module部分实现委托到InternalModule内部类
        ...
    } m_module;
};
```

这个模式核心所在是在头文件通过前向声明`class Impl`，并通过指针指向实例，然后在cpp中定义并实现`Impl`。
```cpp
// Person.h
#include <memory>
class Person {
public:
    Person(std::string name); // (1)
    ~Person(); // (2)
    void Dosomething();
    ...
private:
    struct Impl; // (3)
    std::unique_ptr<Impl> pImpl; // (4)
};
// Person.cpp
#include "Person.h"
class Person::Impl { // impl定义与实现
public:
    Impl(std::string name): name(name) { }
    void Dosomething() {
        // ...
    }
private:
    std::string name;
};
void Person::Dosomething() {
    return pImpl->Dosomething();
}
// 构造、析构函数必须放在实现cpp里，避免inline constructor/destructor导致的incomplete type `Impl'问题
Person::Person(std::string name):
    pImpl(new Impl{name})  // 编程规范问题，可以放到Init()函数中初始化
    { }
Person::~Person() = default;
```

需要注意几个点，由于采用`std::unqiue_ptr`托管`pImpl`，而`std::unqiue_ptr`的声明如下：
```cpp
template <typename _Tp, typename _Dp = default_delete<_Tp>>
class unique_ptr;
```

其中`default_delete`模板实例化需要看到`Impl`的定义，而其定义在cpp中，导致构造、析构函数必须得放到cpp中：
- (1) 构造函数需要在头文件中声明，cpp中实现，而初始化`pImpl`可以放到Init函数中
- (2) 析构函数同上
- (3) 在类内前向声明`Impl`，避免暴露实现出去
- (4) 使用`std::unique_ptr`托管`pImpl`。
