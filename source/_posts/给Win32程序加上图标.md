title: 给Win32程序加上图标
toc: true
original: true
date: 2016-04-24 14:28:55
tags:
- 经验
categories:
- 编程
---

之前写了个[滑动拼图](https://raw.githubusercontent.com/netcan/SlidePuzzle/master/dist/slidepuzzle_windows_release_1.1.zip)游戏，编译出来的`Win32`可执行文件是没有图标的：![http://7xibui.com1.z0.glb.clouddn.com//2016/04/24/QQ%E6%88%AA%E5%9B%BE20160424120224.png](http://7xibui.com1.z0.glb.clouddn.com//2016/04/24/QQ%E6%88%AA%E5%9B%BE20160424120224.png)

查了下资料，增加了图标：

![http://7xibui.com1.z0.glb.clouddn.com//2016/04/24/QQ%E6%88%AA%E5%9B%BE20160424123454.png](http://7xibui.com1.z0.glb.clouddn.com//2016/04/24/QQ%E6%88%AA%E5%9B%BE20160424123454.png)

下面说下具体步骤：
<!--more-->
### 创建rc文件
首先创建`.rc`文件，添加如下内容：

```
id ICON "slidepuzzle.ico"
VERSIONINFO
FILEVERSION     1,1,0,0
PRODUCTVERSION  1,1,0,0
BEGIN
  BLOCK "StringFileInfo"
  BEGIN
    BLOCK "040904E4"
    BEGIN
      VALUE "CompanyName", "Netcan Soft"
      VALUE "FileDescription", "Sliding Puzzle"
      VALUE "FileVersion", "1.1"
      VALUE "InternalName", "slidepuzzle"
      VALUE "LegalCopyright", "Netcan Soft"
      VALUE "OriginalFilename", "slidepuzzle.exe"
      VALUE "ProductName", "SlidePuzzle"
      VALUE "ProductVersion", "1.1"
    END
  END

  BLOCK "VarFileInfo"
  BEGIN
    VALUE "Translation", 0x409, 1252
  END
END
```

根据相关字段进行修改即可，`ICON`指定了图标。
### 导出.res文件
	windres my.rc -O coff -o my.res

### 编译可执行文件
	g++ -o my_app obj1.o obj2.o my.res
