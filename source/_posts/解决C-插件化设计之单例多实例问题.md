---
title: 解决C++插件化设计之单例多实例问题
date: 2019-11-16 15:31:47
tags:
- C/C++
categories:
- 编程
---

### 背景
在插件化设计中，一般会把各个具体产品实现到动态库中，然后通过加载动态库（`dlopen/dlsym`），再通过简单工厂模式创建具体的产品。

实现产品的时候需要看到对应的工厂，才能把创建自己的方法注册到工厂里。

这里的工厂应该采用单例模式，那么加载动态库的时候会出现多份单例的情况。

### 实际情况
测试例子如下结构。
```sh
.
├── CMakeLists.txt
├── main.cpp
├── plugin1.cpp
├── plugin2.cpp
└── singleton.h
```

先来看看makefile，创建了两个so，和一个main。main动态加载了这两个so，并调用so中的函数，让其打印单例的地址:
```makefile
cmake_minimum_required(VERSION 3.16)
PROJECT(test_singleton)
set(CMAKE_CXX_COMPILER clang++)
add_library(plugin1 SHARED plugin1.cpp)
add_library(plugin2 SHARED plugin2.cpp)
add_executable(main main.cpp)
target_link_libraries(main dl)
```

singleton.h:
```cpp
#include <cstdio>
class Singleton {
public:
    static Singleton& Instance()
    {
        static Singleton h;
        return h;
    }
private:
    Singleton() {
        printf("Singleton ctor!\n");
    }
};
```

plugin1.cpp:
```cpp
#include "singleton.h"
#include "stdio.h"
extern "C" void f()
{
    printf("%s: %p\n", __func__, &Singleton::Instance());
}
```

plugin2.cpp:
```cpp
#include "singleton.h"
#include "stdio.h"
extern "C" void g()
{
    printf("%s: %p\n", __func__, &Singleton::Instance());
}
```

main.cpp:
```cpp
#include <cstdio>
#include <dlfcn.h>
int main(int argc, char** argv)
{
    typedef int F();
    void* fd = dlopen("./libplugin1.so", RTLD_LAZY);
    F* f = (F*)dlsym(fd, "f");
    f();
    fd = dlopen("./libplugin2.so", RTLD_LAZY);
    f = (F*)dlsym(fd, "g");
    f();
    return 0;
}
```

运行这个例子：
```sh
$ mkdir build
$ cd build
$ cmake ..
$ make -j
$ ./main
Singleton ctor!
f: 0x7fde8dcd7061
Singleton ctor!
g: 0x7fde8dad5061
```

通过输出可以看到存在多份单例，显然不符合我们预期，然而这个例子用g++编译后（在cmake中将`CMAKE_CXX_COMPILER`设置为g++），运行结果符合我们预期：
```sh
$ make
$ ./main
Singleton ctor!
f: 0x7f18c1b58069
g: 0x7f18c1b58069
```

而如果不通过动态加载的方式（`dlopen/dlsym`），直接将这两个so链接到main中调用，则只有一份单例出现。

综上所述，有如下3种情况：

1. 采用动态加载方式，clang++会出现单例多实例情况(**2020/05/30更新：dlopen通过RTLD_GLOBAL标记解决单例多实例问题！！**)
2. 采用动态加载方式，g++是真正的单例
3. 采用编译链接方式，clang++/g++都是是真正的单例

现在来仔细分析一下如上三种情况，在`singleton.h`中，我们把函数实现（`Singleton::Instance`）在头文件中，然后两个so都include这个头文件，会导致两个so中会有同样的Singleton实例，那为何g++编译出来的单例却是正确的呢？

先来看看通过clang++编译出来的so符号表：
```sh
$ nm -CD libplugin1.so
...
0000000000201068 V guard variable for Singleton::Instance()::h
00000000000009a0 W Singleton::Instance()
0000000000000a20 W Singleton::Singleton()
0000000000201061 V Singleton::Instance()::h
```

再看看通过g++编译出来的so符号表：
```sh
$ nm -CD libplugin1.so
...
0000000000201070 u guard variable for Singleton::Instance()::h
0000000000000a11 W Singleton::Instance()
0000000000000a98 W Singleton::Singleton()
0000000000201069 u Singleton::Instance()::h
```

两者差异是什么呢？可以看到`Singleton::Instance()::h`那行不一样，推测应该是关键所在，一个是`V`，一个是`u`，啥意思呢？参考手册容易得知[https://linux.die.net/man/1/nm](https://linux.die.net/man/1/nm)：

> **"u"**: The symbol is a unique global symbol. This is a GNU extension to the standard set of ELF symbol bindings. For such a symbol the **dynamic linker will make sure that in the entire process there is just one symbol with this name and type in use**.
> **"V"|"v"**: The symbol is a weak object. When a weak defined symbol is linked with a normal defined symbol, the normal defined symbol is used with no error. When a weak undefined symbol is linked and the symbol is not defined, the value of the weak symbol becomes zero with no error. On some systems, uppercase indicates that a default value has been specified.

可以看出u是GNU的扩展，动态链接器可以保证在整个进程生命周期内符号唯一，这就解释了为何g++编译出来的单例正常。而V就是普通的弱符号，没有保证唯一性，在动态加载情况下V与V之间各用各的。

但是又不能直接采用g++编译器，毕竟整个安卓项目的toolchains是clang的，而采用编译链接方式又不够灵活，插件只能做成动态加载的。

### 解决之道
既然问题出在多个so都有同样的`Singleton::Instance()`代码导致动态加载出现多份单例，那么就得采用声明与实现分离的方式，将函数定义到cpp中，声明放在头文件中：

singleton.h
```cpp
#include <cstdio>
class Singleton {
public:
    static Singleton& Instance();
private:
    Singleton() {
        printf("Singleton ctor!\n");
    }
};
```

singleton.cpp
```cpp
#include "singleton.h"
Singleton& Singleton::Instance() {
    static Singleton h;
    return h;
}
```

同时在cmake中添加一个单独的singleton动态库存放真正的实现，其他两个plugin动态库只是简单地引用单例：

```makefile
add_library(singleton SHARED singleton.cpp)
add_executable(main main.cpp)
target_link_libraries(main dl singleton)
```

来看看运行结果，符合我们预期：
```sh
$ make
$ ./main
Singleton ctor!
f: 0x7f8ac0631060
g: 0x7f8ac0631060
```

再来看看`plugin.so`的符号表，由于包含的头文件没有实现`Singleton::Instance`，所以只是简单的引用：
```sh
0000000000000560 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 U printf
                 U Singleton::Instance()
```

看看`singleton.so`的符号表，实现在这：
```sh
0000000000000880 T Singleton::Instance()
00000000000008f0 W Singleton::Singleton()
```

### 扩展：单例模板
前面情况比较简单，一般单例会采用模板方式实现，而模板必须得在头文件中看到实现（C++语言约束），那就不能简单地将`Instance()`分离到cpp中了，不过C++11支持模板显式实例化explicit instantiation，可以将实现分离到cpp中，并显示实例化需要的类型。

singleton.h:
```cpp
#include <stdio.h>
struct Test {
    Test() {
        printf("%s ctor\n", __func__);
    }
};
template<typename T>
class Singleton {
public:
    static T& Instance(); // 实现放到cpp，并显式实例化需要的类型
private:
    Singleton() {}
};
```

singleton.cpp:
```cpp
#include "singleton.h"
template<typename T>
T& Singleton<T>::Instance() {
    static T h;
    return h;
}
template class Singleton<Test>; // 显式实例化需要的类型
```

然后和之前一样，将单例编成单独的so，链接到main中，跑出结果如下，符合预期：
```sh
Test ctor
f: 0x7f883ac1f059
g: 0x7f883ac1f059
```

而`plugin.so`的符号表也是引用单例，真正的单例在`singleton.so`中了：
```sh
0000000000000588 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 U printf
                 U Singleton<Test>::Instance()
```

`singleton.so`:
```sh
0000000000201060 V guard variable for Singleton<Test>::Instance()::h
0000000000000a20 W Test::Test()
00000000000009a0 W Singleton<Test>::Instance()
0000000000000a50 W Singleton<Test>::Singleton()
0000000000000a50 W Singleton<Test>::Singleton()
0000000000201059 V Singleton<Test>::Instance()::h
```

所以实际项目中，可以把工厂这个单例放到单独的一个so中，并链接到main模块中，其他产品采用动态加载方式，注册创建方法到对应工厂单例中。
