---
title: 设计并实现一个拓扑排序DSL
toc: true
original: true
date: 2020-08-15 14:01:08
tags:
- C/C++
- 元编程
categories:
- 编程
---

## 背景
最近在重构项目，项目主要和图有关，图描述了各个节点的连接关系，这些连接关系有条件选中，图也会局部变化，而各个具体节点则承载了实际业务。

由于之前的设计的代码通过可视化工具生成表，将连接的边和条件都存到一个表中，运行时根据条件选中节点和边，从而拼凑成一幅完整的图。而这部分代码比较难读，各种容器遍历迭代，才得到最终的图数据结构，而想要知道最终图长啥样，需要运行时抓日志获得。

考虑图结构是静态的，只不过连接条件是动态的，那么拼凑成图的过程可以在编译期完成，由于元编程计算得到的类型保存了整个图的连接关系，而类型不耗费额外内存，真正做到零成本抽象，并且还能做些额外检查。

既然用C++语言了，就应该发挥语言的价值，这样图那部分代码也会更加直观。

于是我们设计了一套DSL，通过DSL来描述一副图，图的连接关系也能有条件，本文为了简单起见，这里记录一下我设计实现的用于解决**拓扑排序**问题的Demo，虽整个IDEA比较粗糙简陋，但大方向确定了，更多细节需要在项目中逐步实现。

## 用户界面
我设计的TaskDSL的用户界面如下，整个拓扑关系在编译期计算完成：
```cpp
__task(__job(Plating) -> __job(Servce1),
       __job(Meat)    -> __job(Plating),
       __job(Plating) -> __job(Servce2),
       __job(Garnish) -> __job(Plating)).Run();
```

而各个Job的实现如下：
```cpp
void Meat()
{ puts("肉..."); }
void Garnish()
{ puts("配菜..."); }
void Plating()
{ puts("装配..."); }
void Servce1()
{ puts("送菜1..."); }
void Servce2()
{ puts("送菜2..."); }
```

输出结果如下：
```sh
肉...
配菜...
装配...
送菜1...
送菜2...
```

## 实现
观察整套DSL，无非就是描述各个Job的连接关系，而C++提供的函数后置返回类型箭头`->`很好的用于表达连接。

首先是`__task`原语，`Task`类的封装，保存了边集，提供了一个运行环境`Run`接口：
```cpp
#define __task(...) Task<__VA_ARGS__>{}
```

而`__job`原语主要用`JobWrapper`包装了一层，供后续扩展用，同时也可以记录一些额外信息：
```cpp
#define __job(job) auto(*) (JobWrapper<job>)
template<auto J>
struct JobWrapper {
    void operator()() {
        return J();
    }
};
```

那么`__job(Meat) -> __job(Plating)`展开后生成如下代码：
```cpp
auto(*) (JobWrapper<Meat>) -> auto(*) (JobWrapper<Plating>)
```

如何获取源`Job`和目的`Job`呢？写一个`Connection`模板萃取一下：
```cpp
template<typename LINK>
struct Connection {};
template<typename Job1, typename Job2>
struct Connection<auto(*)(Job1) -> auto(*)(Job2)> {
    using FROM = Job1;
    using DST = Job2;
};
template<typename Con>
struct GetFrom { using type = typename Con::FROM; };
template<typename Con>
struct GetDst { using type = typename Con::DST; };
```

`Connection<LINK>::FROM`和`Connection<LINK>::DST`分别得到对应信息。

来看看`Task`的声明，输入边集：
```cpp
template <typename... LINKS>
class Task;
```

将输入的边集存到类型中，并获取所有的Job，最后去重，关于Map/Filter/Unique/Find/Sort实现可以参考我的 [TypeList.hpp](https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/Typelist.hpp)：
```cpp
using Connections = TypeList<Connection<LINKS>...>;
using FromJobs    = typename Map<Connections, GetFrom>::type;
using Jobs        = typename Map<Connections, GetDst, FromJobs>::type;
using AllJobs     = typename Unique<Jobs>::type;
static constexpr size_t JobNums = AllJobs::size;
```

下一步是查找一个Job的**直接**后继Job列表，用于构造邻接表。算法很简单，根据输入的Job和Connections，通过Filter操作过滤出From和该Job一致的Connections，并对得到的Connections做Map操作，取出Dst Job List：
```cpp
template<typename Job>
class FindJobDependencies {
    template<typename C> struct Dependency
    { static constexpr bool value = std::is_same_v<typename GetFrom<C>::type, Job>; };
    using JobDependencies = typename Filter<Connections, Dependency>::type;
public:
    using type = typename Map<JobDependencies, GetDst>::type::template prepend<Job>; // prepend FROM node to result front
};
```

对得到的Dst Job List做Map操作，得到所需要的新类型（邻接表）：
```cpp
template <typename DEP>
struct Dependency {
    struct type {
        using Job = typename DEP::head;
        using Dependencies = typename DEP::tails;
    };
};
using JobDependenciesMap = typename Map<
        typename Map<AllJobs, FindJobDependencies>::type, // Dst Job List of List
        Dependency>::type;
```

而JobDependenciesMap就是我们需要的邻接表。

有了节点的**直接**后继还不够，我们需要把**间接**后继节点也加入，这样才方便做拓扑排序。算法首先展开一个Job的直接后继节点列表，对后继中的每个Job，需进一步递归展开其直接后继并去重，直到终端节点，结束递归。期间也很容易做环的检测，这里我没有做，留给读者思考。整个算法其实是一个Fold操作，由于我还没实现Fold，这里就直接递归展开了。更多细节见注释。
```cpp
template<typename DEPS, typename OUT = TypeList<>, typename = void>
struct FindAllDependencies {
    using type = OUT; // 边界case，结束递归时直接输出结果
};
template<typename DEPS, typename OUT>
class FindAllDependencies<DEPS, OUT, std::void_t<typename DEPS::head>> {
    using J = typename DEPS::head; // 取出后继表中第一个Job
    template <typename DEP> struct JDepsCond
    { static constexpr bool value = std::is_same_v<typename DEP::Job, J>; };
    using DepsResult = typename FindBy<JobDependenciesMap, JDepsCond>::type; // 从邻接表查找Job的后继节点列表
        using JDeps = typename Unique<typename DepsResult::Dependencies>::type; // 去重操作
public:
    using type = typename FindAllDependencies<
        typename JDeps::template exportTo<DEPS::tails::template append>::type,
        typename OUT::template appendTo<J>>::type; // 将得到的后继列表合并，进一步递归展开，并输出当前Job到列表
};
```

和之前一样做Map操作，构造出邻接表JobAllDependenciesMap，其包含节点的所有直接、间接后继。
```cpp
template<typename DEP>
struct FindJobAllDependencies {
    struct type {
        using Job = typename DEP::Job;
        using AllDependencies = typename FindAllDependencies<typename DEP::Dependencies>::type;
    };
};
template<typename DEP>
struct GetJob { using type = typename DEP::Job; };
using JobAllDependenciesMap = typename Map<JobDependenciesMap, FindJobAllDependencies>::type;
```

有了这个表，我们就可以实现比较函数，定义`节点LHS < 节点RHS`，则`节点LHS->节点RHS`，即`节点RHS`是`节点LHS`的后继。
```cpp
template<typename LHS, typename RHS>
struct JobCmp {
    static constexpr bool value =
        Elem<typename LHS::AllDependencies, typename RHS::Job>::value;
};
```

对JobAllDependenciesMap进行排序，并对结果做Map操作，取出Job节点，从而得到最终的排序的Job List结果：
```cpp
using SortedJobs = typename Map<typename Sort<JobAllDependenciesMap, JobCmp>::type, GetJob>::type;
```

最后就是实现`Task::Run`接口，按照拓扑序执行Job。首先将Sorted Job List整合到一块去，才能依次执行：
```cpp
template<typename ...Jobs>
struct Runable { };
template<auto ...J>
struct Runable<JobWrapper<J>...> {
    void operator()() {
        return (JobWrapper<J>{}(),...);
    }
};
```

最终将Sorted Job List作为模板参数导出到`Runable`并实例化运行：
```cpp
void Run() {
    using Jobs = typename SortedJobs::template exportTo<Runable>;
    return Jobs{}();
}
```

以上所有细节都放在了这，需要的话可以进一步阅读。[TaskDSL.cpp](https://github.com/netcan/recipes/blob/master/cpp/metaproggramming/TaskDSL.cpp)
## 心得
以上就是实现一个TaskDSL所需要的细节，从中可以看出C++模板元编程的威力，只不过被我们忽略掉了。很多东西都不新鲜了，源自于函数式编程，只不过C++模板语法过于复杂。为了可维护性考虑，可以借鉴[hana](https://www.boost.org/doc/libs/1_61_0/libs/hana/doc/html/index.html)这个元编程库，用起来就和普通函数一样，将操作类型转换成操作值。
