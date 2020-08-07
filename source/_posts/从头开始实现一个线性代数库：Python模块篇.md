---
title: 从头开始实现一个线性代数库：Python模块篇
toc: true
top: false
original: true
date: 2018-05-29 17:56:03
tags:
- 数学
- 线性代数
- C/C++
- Python
categories:
- 编程
---

这两天用C/C++实现了一下线性代数库的Python模块，大部分操作已经封装完成，剩下的慢慢补坑吧= =

关于线性代数库的一些算法实现，可以参考我的前一篇文章[从头开始实现一个线性代数库：算法实现篇](http://www.netcan666.com/2018/05/27/%E4%BB%8E%E5%A4%B4%E5%BC%80%E5%A7%8B%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E7%BA%BF%E6%80%A7%E4%BB%A3%E6%95%B0%E5%BA%93%EF%BC%9A%E7%AE%97%E6%B3%95%E5%AE%9E%E7%8E%B0%E7%AF%87/)。现在主要总结一下如何用C/C++编写Python模块，代码地址[https://github.com/netcan/LinAlg/tree/master/linalg.module](https://github.com/netcan/LinAlg/tree/master/linalg.module)。

先来看看效果：
```python
In [1]: ls
linalg.cpython-36m-darwin.so*

In [2]: import linalg

In [3]: m = linalg.Matrix([
   ...:     [43, 63, 57, 35],
   ...:     [63, 26, 32, 35],
   ...:     [57, 32, 78, 76],
   ...:     [35, 35, 76, 25],
   ...: ])

In [4]: m.Jacobi()
Out[4]: C(-19.248723 33.225382 197.775875 -39.752533)

In [5]: m.det()
Out[5]: 5028170.999999999
```

编写模块最麻烦的部分应该是引用计数了，如果能够妥善处理，将会事半功倍，我因为这个问题debug好久了。紧接着就是异常处理了。主要参考资料还是官方文档：
- [Extending Python with C or C++](https://docs.python.org/3/extending/extending.html)
- [Defining Extension Types: Tutorial](https://docs.python.org/3/extending/newtypes_tutorial.html)
- [Defining Extension Types: Assorted Topics](https://docs.python.org/3/extending/newtypes.html)
- [Building C and C++ Extensions](https://docs.python.org/3/extending/building.html)

## 定义模块
首先定义一个名为`linalg`的模块，其下有两个类：`linalg.Vector`和`linalg.Matrix`。

```cpp
static PyModuleDef linalgmodule = {
    PyModuleDef_HEAD_INIT,
    "linalg",   /* name of module */
    NULL,		/* module documentation, may be NULL */
    -1,			/* size of per-interpreter state of the module,
                 or -1 if the module keeps state in global variables. */
};
```

如果需要定义一些模块级方法，需要在定义`linalgmodule`的时候指定`m_methods`，这里不需要这类方法，就省略了。

接下来初始化模块，定义名为`PyInit_linalg`的函数：
```cpp
PyMODINIT_FUNC
PyInit_linalg(void) {
    PyObject *m;
    m = PyModule_Create(&linalgmodule);
    if (m == NULL) return NULL;
    // 添加Vector/Matrix类
    if (PyType_Ready(&PyVectorType) < 0) return NULL;
    if (PyType_Ready(&PyMatrixType) < 0) return NULL;
    Py_INCREF(&PyVectorType);
    Py_INCREF(&PyMatrixType);
    PyModule_AddObject(m, "Vector", (PyObject*)&PyVectorType);
    PyModule_AddObject(m, "Matrix", (PyObject*)&PyMatrixType);
    return m;
}
```

可以看到在初始化模块的时候，添加了`linalg.Vector`和`linalg.Matrix`两个类。接下来定义这两个类，以及类方法。

## 定义`linalg.Vector`类
由于`linalg.Matrix`类和`linalg.Vector`类定义的方法基本相同，这里只总结一下`linalg.Vector`类的定义以及实现。

首先定义`VectorObject`对象类：
```
typedef struct {
    PyObject_HEAD;
	Vector ob_vector; // 需要封装的成员
} PyVectorObject;
```

接着是`VectorType`类的定义：
```
static PyTypeObject PyVectorType = {
    PyObject_HEAD_INIT(NULL)
    .tp_name = "linalg.Vector", // 模块类名
    .tp_doc = "Vector objects", // doc描述
    .tp_basicsize = sizeof(PyVectorObject), // 对象大小
    .tp_itemsize = 0,
    .tp_dealloc = (destructor) PyLinAlg_dealloc, // 析构函数
    .tp_new = PyType_GenericNew, // 构造函数
    .tp_init = (initproc) PyVector_init, // 初始化函数
    .tp_members = PyVector_members, // 类成员
    .tp_methods = PyVector_methods, // 类方法
    .tp_flags = Py_TPFLAGS_DEFAULT,
    .tp_str = (reprfunc)PyVector_str, // str(obj), print用
    .tp_repr = (reprfunc)PyVector_str,
    .tp_as_sequence = &PyVectorSeq_methods, // 一些序列类方法，例如vec[i]
    .tp_as_number = &PyVectorNum_methods, // 基本运算方法，例如vecA + vecB
};
```

先来看看`linalg.Vector`的析构函数，若不是单例模式，所有类都需要这个函数在引用计数为0的时候来释放对象：
```cpp
static void
PyLinAlg_dealloc(PyObject *self) {
    Py_TYPE(self)->tp_free((PyObject *)self);
}
```

构造函数使用默认的`PyType_GenericNew`就好，初始化一个`Vector`类，在C++接口中是这样的：
```cpp
Vector v({16,22,32,44});
v.show();
```

现在用Python模块方式进行初始化，为了简单起见，这里只支持列表类型初始化，即：
```python
In [6]: v = linalg.Vector([1,2,3,4])

In [7]: v
Out[7]: C(1.000000 2.000000 3.000000 4.000000)
```

初始化函数如下：
```cpp
static int
PyVector_init(PyVectorObject *self, PyObject *args, PyObject *kwds) {
    PyObject *pList, *pItem;
    if (!PyArg_ParseTuple(args, "O!", &PyList_Type, &pList)) { // 初始化参数必须为列表
        PyErr_SetString(PyExc_TypeError, "parameter must be a list.");
        return -1;
    } 
    Py_ssize_t n = PyList_Size(pList); // 获取列表长度
    if(n <= 0) {
        PyErr_SetString(PyExc_TypeError, "size of list must be greater than 0");
        return -1;
    }
    vector<double> data;
    for(Py_ssize_t i = 0; i < n; ++i) {
        pItem = PyList_GetItem(pList, i);
        if(! isNumber(pItem))  return -1; // 列表内的元素必须为数字
        else data.push_back(getNumber(pItem));
    }
    self->ob_vector = Vector(data); // 存入封装Vector的VectorObject类型中
    return 0;
}
```

而对于类支持的方法，需要定义一个数组`PyVector_methods`说明，比如`linalg.Vector`类支持`T()`, `copy()`方法，那么：
```
static PyMethodDef PyVector_methods[] = {
    {"T", (PyCFunction)PyVector_T, METH_NOARGS, "change vector type"},
    {"copy", (PyCFunction)PyVector_Copy, METH_NOARGS, "deep copy of vector"},
    {NULL}  /* Sentinel */
};
```

然后实现`PyVector_T()`和`PyVector_Copy`即可。

而对于一些magic方法，例如`__add__`, `__getitem__`等等，需要在`PyTypeObject`的`.tp_as_number`和`.tp_as_sequence`数组中指明。

### 引用计数导致的BUG
之前因为返回对象的引用计数导致了非常麻烦的BUG，这里贴出来一下：
```
static PyObject *
PyVector_imul(PyVectorObject *self, PyObject *arg) {
    if(!arg || ! isNumber(arg)) return NULL;
    Py_XINCREF(self); // 不加这个会崩溃...因为返回的对象没有被接收会被释放掉
    self->ob_vector *= getNumber(arg);
    return (PyObject*)self;
}
```

这里定义了一下`*=`类方法，当执行`v *= 5`等语句，可能导致崩溃，需要增加引用计数来获得所有权，因为操作的对象是*borrowed references*的，那么返回的时候由于没有被其他对象引用，将被释放，而后来要是有对象申请空间也用到这块内存的话，就会出现异常。

官方文档是这么说明的，对于[PyNumber_InPlaceAdd](https://docs.python.org/3/c-api/number.html#PyNumber_InPlaceAdd)这类方法应该返回的是新引用，而不是*borrowed references*。

> PyObject* PyNumber_InPlaceAdd(PyObject *o1, PyObject *o2)
> Return value: **New reference.**
> Returns the result of adding o1 and o2, or NULL on failure. The operation is done in-place when o1 supports it. This is the equivalent of the Python statement o1 += o2.

在StackOverflow上有这么一个回答：[https://stackoverflow.com/questions/11897597/implementing-nb-inplace-add-results-in-returning-a-read-only-buffer-object](https://stackoverflow.com/questions/11897597/implementing-nb-inplace-add-results-in-returning-a-read-only-buffer-object)

> According to the documentation, the inplace add operation for an object returns a new reference.
> By returning self directly without calling Py_INCREF on it, your object will be freed while it is still referenced. If some other object is allocated the same piece of memory, those references would now give you the new object.

## 编译、链接模块
掌握了如何定义模块、类，剩下的就简单了，最后是通过编写`setup.py`来生成模块，如下：
```python
from distutils.core import setup, Extension

linalgmodule = Extension('linalg',
        extra_compile_args = ['-std=c++14'],
        sources = ['src/linalgmodule.cpp'],
        include_dirs = ['include'],
        libraries = ['linalg'],
        )

setup (name = 'linalg',
        version = '0.1',
        author = 'netcan',
        author_email = 'netcan1996@gmail.com',
        description = 'A Linear Algebra library for studying by netcan',
        ext_modules = [linalgmodule],
        )
```

编译的时候，通过如下编译，
```bash
 python setup.py build
 ```

 可能会报错，例如找不到`std::move`，这时候需要指定标准头文件的位置了：
```bash
CFLAGS='-I/Library/Developer/CommandLineTools/usr/include/c++/v1/' python setup.py build
 ```
