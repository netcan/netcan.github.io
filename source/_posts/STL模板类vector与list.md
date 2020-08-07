title: STL模板类vector与list
toc: true
original: true
tags:
  - C/C++
  - 算法
  - 数据结构
categories:
  - 编程
date: 2015-04-23 18:36:44
updated: 2015-04-23 18:36:44
---

在学校借了本＜数据结构＞感觉蛮好的，想买一本可是看了差不多了，那还是做好笔记吧．

C++的STL有很多与线性表相关的容器，如向量vector，列表list等顺序容器．

### 向量

向量是最常用的容器，STL中向量是使用数组来实现的，因此向量具有顺序表的所有特点，可以快速随机访问存取任意元素．向量是同一种数据类型的对象的集合，每个对象根据其位置有一个整数索引与其对应，类似数组．

在使用向量前，必须包含对应的头文件：

	#include <vector>
	using std::vector;
需要注意的是vector是一个类模板，而非数据类型，在定义时必须要说明vector保存的数据对象类型．
<!-- more -->

	vector<int> ivec;    //定义向量对象ivec
	vector<int> ivec1(ivec);    //定义向量对象ivec1，并用ivec初始化
	vector<int> ivec2(n,2);     //定义向量对象ivec２，包含了n个值为i的元素
	vector<int> ivec3(n);       //定义向量对象ivec３，包含了n个值为０的元素
与数组不同的是，vector不需要声明长度．标准库负责管理与储存元素的内存，用户不需要担心长度不够．这也是其重要属性，在运行时可以高效的添加元素．

下面给段例子．

```cpp
/*************************************************************************
    > File Name: vector.cpp
    > Author: Netcan
    > Mail: 1469709759@qq.com
    > Created Time: Thu 23 Apr 2015 17:34:59 CST
 ************************************************************************/

#include<iostream>
#include<vector>
using namespace std;

int main() {
    vector<int> ivec(5, 2);
    for(int i=0; i<ivec.size(); ++i)
        cout << ivec[i]; // 是用下标访问元素
    cout << endl;
    ivec.push_back(3); // 尾部插入元素3
    ivec.insert(ivec.begin(), 1); // 头部插入元素1
    for(vector<int>::iterator i=ivec.begin(); i!=ivec.end(); ++i) // 是用迭代器访问
        cout << *i;
    cout << endl;
    return 0;
}
```
执行结果如下：

	22222
	1222223
vector提供了几个重要的接口来查看或设置分配空间和实际储存元素的数目，这些接口包括size(), capacity(), reserve()等等。

下面给个例子来说明向量的储存状态：

```cpp
/*************************************************************************
    > File Name: vector_stor.cpp
    > Author: Netcan
    > Mail: 1469709759@qq.com
    > Created Time: Thu 23 Apr 2015 18:09:25 CST
 ************************************************************************/

#include<iostream>
#include<vector>
using namespace std;

int main() {
    vector<int> ivec;
    cout << "size: " << ivec.size() << " capacity: " << ivec.capacity() << endl;
    for(int i=0; i<10; ++i)
        ivec.push_back(i);
    cout << "size: " << ivec.size() << " capacity: " << ivec.capacity() << endl;
    for(int i=0; i<90; ++i)
        ivec.insert(ivec.end(), i);
    cout << "size: " << ivec.size() << " capacity: " << ivec.capacity() << endl;
    ivec.reserve(200); // 设置对象实际占用空间
    cout << "size: " << ivec.size() << " capacity: " << ivec.capacity() << endl;

}
```

执行结果：

	size: 0 capacity: 0
	size: 10 capacity: 16
	size: 100 capacity: 128
	size: 100 capacity: 200
### 列表

列表是STL中线性表的链式储存方式，STL标准库中一般采用双向循环链表实现列表。因此list容器使用不连续的空间区域，允许向前和向后逐个遍历元素，但不支持随即访问，查找某个元素时往往需要遍历相邻的若干元素。作为其优点，在任何时刻可以快速的插入和删除，并且不需要移动其他元素。

在使用list之前，需要包含相应的头文件。

	#include <list>
	using std::list;
list容器有很多与vector容器相同的接口。

下面给出C++STL参考网址：

[http://en.cppreference.com/w/cpp/container](http://en.cppreference.com/w/cpp/container)
