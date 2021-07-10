---
title: 一个C++静态反射框架：ConfigLoader
toc: true
original: true
date: 2021-07-10 16:37:11
tags:
- C/C++
- 元编程
categories:
- 编程
---

框架地址：[https://github.com/netcan/config-loader](https://github.com/netcan/config-loader)

相关技术：[如何优雅的实现 C++ 编译期静态反射](/2020/08/01/如何优雅的实现C-编译期静态反射/)

`config-loader`是一个使用C++17编写的 **解析配置文件** 到 **原生数据结构** 的静态反射框架，它拥有如下特点：

- 简单的接口，用户通过 **定义数据结构** 与提供对应的 **配置文件**，框架利用元编程技术生成 **读取** 接口
- 设计符合开闭原则，扩展 数据结构 无需修改框架
- 目前支持XML与JSON格式的配置文件，多种方式可以 **灵活组合**
- 轻量级，容易集成，核心代码不到1000行
- 支持 嵌套的数据结构、STL容器
- 测试用例完备

将来计划：

- 支持Yaml配置文件
- 支持从原生数据结构写到配置文件、打印数据结构
- 通过CMake选项来控制支持的格式
- 提供额外的C++20版本

## 快速上手
使用 `DEFINE_STRUCT` 定义数据结构：

```cpp
// define and reflect a struct
DEFINE_STRUCT(Point,                          // struct Point {
              (double) x,                     //     double x;
              (double) y);                    //     double y;
                                              // };

// vector and string
DEFINE_STRUCT(SomeOfPoints,                   // struct SomeOfPoints {
              (std::string) name,             //     std::string name;
              (std::vector<Point>) points);   //     std::vector<Point> points;
                                              // };
```

提供配置文件，按需加载：

```cpp
SomeOfPoints someOfPoints;
auto res = JsonLoader<SomeOfPoints>().load(someOfPoints, [] {
    return R"(
        {
            "name": "Some of points",
            "points":[
                { "x": 1.2, "y": 3.4 },
                { "x": 5.6, "y": 7.8 },
                { "x": 2.2, "y": 3.3 }
            ]
        }
    )";
});
REQUIRE(res == Result::SUCCESS);
REQUIRE_THAT(someOfPoints.name, Equals("Some of points"));
REQUIRE(someOfPoints.points.size() == 3);
```

又或者，通过XML配置文件。
```cpp
SomeOfPoints someOfPoints;
auto res = XMLLoader<SomeOfPoints>().load(someOfPoints, [] {
    return R"(
        <?xml version="1.0" encoding="UTF-8"?>
        <some_of_points>
            <name>Some of points</name>
            <points>
                <value><x>1.2</x><y>3.4</y></value>
                <value><x>5.6</x><y>7.8</y></value>
                <value><x>2.2</x><y>3.3</y></value>
            </points>
        </some_of_points>
    )";
});
REQUIRE(res == Result::SUCCESS);
REQUIRE_THAT(someOfPoints.name, Equals("Some of points"));
REQUIRE(someOfPoints.points.size() == 3);
```

有时候，你的软件系统需要一个统一的配置管理模块，管理 所有的数据结构 与 对应的配置文件，这时可以通过组合各个 `Loader` 来定义管理者。

```cpp
inline Deserializer ConfigLoaderManager(
    JsonLoader<Point>("/etc/configs/Point.json"_path),
    XMLLoader<Rect>("/etc/configs/Rect.xml"_path),
    JsonLoader<SomeOfPoints>() // 按需提供配置文件
);
```

同样地，使用`load`接口按需加载，`ConfigLoaderManager`会自动根据 配置的路径 与 给定的数据结构 进行解析。你的IDE应该能够获得所有的 `load` 接口。

```text
 82     Deserializer ConfigLoaderManager(
 83             JsonLoader<Point>(),
 84             XMLLoader<Rect>(),
 85             JsonLoader<SomeOfPoints>()
 86     );
 87     ConfigLoaderManager.l
 88                         load(Rect &obj)~                                   f [LS]
 89                         load(Point &obj)~                                  f [LS]
 90 }                       load(SomeOfPoints &obj, GET_CONTENT &&getContent)~ f [LS]
~                           load(SomeOfPoints &obj)~                           f [LS]
~                           load(Rect &obj, GET_CONTENT &&getContent)~         f [LS]
~                           load(Point &obj, GET_CONTENT &&getContent)~        f [LS]
```

## 注意事项
当前框架依赖如下两个库：
- tinyxml2，解析xml配置文件用
- jsoncpp，解析json配置文件用

将来可能通过CMake选项来使能这些库，避免在实际使用中产生不必要的依赖：只用xml就只依赖xml的解析库。

本框架需要配置文件按规范的格式提供。以XML为例，要求字段名与XML标签名对应，值与XML的文本内容对应；对于`map`数据结构，标签通过属性`name`作为Key名。

当前错误码语义。
```cpp
enum class Result {
    SUCCESS,              // 解析成功
    ERR_EMPTY_CONTENT,    // 解析文件为空
    ERR_ILL_FORMED,       // 解析文件非法
    ERR_MISSING_FIELD,    // 丢失字段
    ERE_EXTRACTING_FIELD, // 解析值失败
    ERR_TYPE,             // 类型错误
};
```
