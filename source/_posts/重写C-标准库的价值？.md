---
title: 重写C++标准库的价值？
toc: true
original: true
date: 2022-03-12 09:55:19
tags:
- C/C++
categories:
- 编程
---
众所周知，标准库中的算法容器是普通人很难手写超越的，因为这归功于C++的模板、编译时计算特性，它拥有零成本抽象能力，也就是说无论使用模板机制做多少层抽象，最后生成的代码和手写C代码一样高效，这就是为何C++相对于C来说拥有**易用**的接口，并且不会导致性能损失。

但本文章的主题不在于模板编程，而在于探讨重写标准库的价值。在这之前需要声明下，C++标准中自定义了标准库的接口，以及每个接口的时间复杂度，具体实现由各大编译器厂商提供。此外替换标准库的一些容器，即便接口不完全一致，也符合本文讨论的范畴。

以链表为例，Linux C语言的经典做法是：
```c
struct list_head {
    struct list_head *next, *prev;
};

struct io_buffer {
    struct list_head list;
    __u64 addr;
    __u32 len;
    __u16 bid;
};
```

对链表的操作则是由各种宏实现：
```c
#define list_entry(ptr, type, member) container_of(ptr, type, member)
#define list_for_each_entry(pos, head, member)               \
    for (pos = list_first_entry(head, typeof(*pos), member); \
        !list_entry_is_head(pos, head, member);              \
        pos = list_next_entry(pos, member))
```


而C++标准库中的链表方式则是这样：
```cpp
struct io_buffer {
    __u64 addr;
    __u32 len;
    __u16 bid;
};
std::list<io_buffer> ls;
```

这两种方式的区别在哪？唯一区别在于 `io_buffer` 元素本身是否携带了链表的指针信息，Linux方式则内嵌 `list_head` 于元素中，C++标准容器的指针信息在哪？

C++标准容器的链表指针信息则是由 `std::list` 维护，也就是说只有在元素被插入到链表的时候，将发生一次内存分配，这包含了额外的链表指针信息 + 元素本身的内存，然后将元素的内容通过拷贝、移动构造到分配出来的节点。

也就是说同样是链表，对于插入操作，C++标准容器多了 **一次内存分配** +一次移动、拷贝构造的开销，其中性能的关键在于那次**内存分配动作**。而C语言由于**元素本身**携带了链表节点信息，插入动作只是指针的调整。

有人指出使用 `std::list<io_buffer *>`，性能就和Linux C的链表差不多了。但这只能省移动、拷贝构造开销，省不了额外指针节点的内存分配，关键在于内存分配操作。

这并不是C++的问题，这是两种不同的思路： *侵入式* 与 *非侵入式* 链表。侵入式即要求用户定义元素时额外定义链表的节点信息，而非侵入式（C++标准库）则没有这个要求，那么就需要额外分配外挂的信息。

我尝试用C++实现了侵入式链表方案，通过性能测试可以得到如下数据（ARM）：

|  ns/op |           op/s |  err% | ins/op | total | benchmark
|-------:|---------------:|------:|-------:|------:|:----------
| 253.62 |   3,942,909.94 | 22.9% | 247.25 |  0.00 | `std::forward_list push_front`
| 208.03 |   4,806,889.46 |  3.7% | 249.08 |  0.00 | `std::forward_list push_back`
|   2.03 | 492,790,574.13 |  0.0% |   5.00 |  0.00 | `SList PushFront`
|   5.46 | 183,296,000.00 |  0.0% |   7.00 |  0.00 | `List PushBack`

`std::list` 和 `std::forward_list` 插入一个节点需要 200 到 250ns，而侵入式链表插入一个节点只需要 2 到 5 ns，关键在于动态内存分配比较耗时。在内核中大量依赖链表、树结构，这体现了侵入式方案的优越性。

侵入式方案相对于非侵入式方案在于接口的 **易用性**，这是牺牲易用性换取性能的优势的一个例子。

C++设计与标准库也未必全是侵入式的，例如智能指针便提供了两种方案，当用户类继承自`enable_share_from_this`时，该类就是侵入式智能指针，引用计数和类融为一体，能够将裸指针提升至智能指针；而用户不继承该类时，则是非侵入式智能指针，构造它需要为引用计数块额外分配一次内存（除非用户使用`make_shared`构造，节省引用计数块的额外内存分配），由于引用计数块和用户类内存分离，也就不允许裸指针到智能指针的转换。

C++的虚表设计也是侵入式的，如果用户需要表现运行时多态性，则需要继承并实现虚接口，这样虚表和用户类便融为一体。在C++17时标准库则引入 `std::visit` 和 `std::variant`，通过它也能实现运行时多态，并且由元编程技术生成虚表，换句话说虚表是外挂用户类的，调用时生成的虚表信息可以直接使用调用栈内存，也就无需额外为虚表分配内存，性能上也不会有什么损失。那它们的区别就在于使用上了，前者对类扩展开放，对行为扩展封闭；后者对类扩展封闭，对行为扩展开放。

在链表数据结构中我们看到了侵入式方案性能上的优越性，那么对于一些树结构，有没有性能优势呢？笔者花了两天时间手撸 **红黑树** 后，经过测试发现性能略逊于标准库的`std::set`，插入一个节点侵入式红黑树需要250ns，而 `std::set`只需要200ns，删除略快些，12ns vs 55ns（x86-64）。

是不是笔者实现的太挫了？经过一番搜索，C++的侵入式容器库中实现了 `set` 只有 Boost.Intrusive 一家，于是尝试使用它做性能测试，得到如下性能测试数据（macOS）：

|   ns/op |          op/s |  err% | total | benchmark
|--------:|--------------:|------:|------:|:----------
|  170.48 |  5,865,909.42 |  8.6% |  1.79 | `std::multiset insert`
|   94.64 | 10,566,395.80 | 34.7% |  0.98 | `std::multiset erase`
|  240.30 |  4,161,400.76 |  6.8% |  2.41 | `intrusive::multiset insert`
|   17.44 | 57,345,039.37 |  1.8% |  0.21 | `intrusive::multiset erase`

看来 Boost.Intrusive 的实现比我的还挫啊。因此对于树而言，侵入式方案没有有优势，这时候建议使用 `std::set` 即可，因为很难超越它的实现了。

难怪笔者阅读数个调度器的实现（例如 Executors）都只实现了侵入式链表，而没有实现侵入式树，看来大佬们已经知道了这个事实，而且 Boost.Intrusive 谈及性能时，也只提链表。C++社区其他的侵入式库都没有提供侵入式红黑树的实现。

然而链表在应用开发中很少用，除了调度器场景中会需要，他们大多数的情况下可以被更全能的vector替代，因为push数据时无需频繁地分配内存，而链表每次插入数据都需要分配一次内存，这也是为何 `std::queue` 和 `std::stack` 默认用 `std::deque` 容器实现，也是出于 **内存分配频率** 的考虑。

而vector重写的价值不大，因为他们已经在大多数情况下表现够优秀，如果非要重写，那么可以参考一些常用的思路，例如SmallVector，在只有几个元素的时候仅使用调用栈的内存，一旦元素过多可以便使用堆内存，即便LLVM/Clang等项目重写了标准库，但仍然普遍使用std标准库，只有那极少数性能关键的地方使用。

C++标准明确定义了各种数据结构操作的时间复杂度，因此对这些操作的重写价值不大。综上所述，**频繁的内存分配** 操作是性能杀手，因此很多重写标准库的方案都是想法设法避免频繁的内存分配动作，有时候重写allocator的价值可能远比重写整个数据结构要大得多。

在嵌入式场景中内存通常是确定的，它们避免动态内存分配，标准库实现要求异常安全，这会给二进制文件带来一定的大小开销，这时候考虑重写标准库是值得的。

Rust有本书专门教你怎么写链表，最终的结论就是尽量不要用链表而是用Vec，有这精力都能把其他数据结构和算法吃透了：

> but all of these cases are super rare for anyone writing a Rust program. 99% of the time you should just use a Vec (array stack), and 99% of the other 1% of the time you should be using a VecDeque (array deque). These are blatantly superior data structures for most workloads due to less frequent allocation, lower memory overhead, true random access, and cache locality.

那么，最后重写标准库的价值留给读者判断。

参考资料：

- [https://rust-unofficial.github.io/too-many-lists/](https://rust-unofficial.github.io/too-many-lists/)
- [https://www.boost.org/doc/libs/1_78_0/doc/html/intrusive.html](https://www.boost.org/doc/libs/1_78_0/doc/html/intrusive.html)

