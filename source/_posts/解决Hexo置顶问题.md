title: 解决Hexo置顶问题
toc: false
original: true
date: 2015-11-22 15:54:30
tags:
- Hexo
- Nodejs
categories:
- 随笔
top: true
---
2017-10-25 15:44更新，为了防止每次更新、安装都要修改代码，现在可以直接从仓库里安装了：

```
$ npm uninstall hexo-generator-index --save
$ npm install hexo-generator-index-pin-top --save
```

然后在需要置顶的文章的`Front-matter`中加上`top: true`即可。

Github: [https://github.com/netcan/hexo-generator-index-pin-top](https://github.com/netcan/hexo-generator-index-pin-top)

---

考虑到之前的博客有置顶文章，所以需要置顶功能。

`Google`了一下解决方案，发现了本博客主题是支持置顶功能的，在需要置顶的`Front-matter`中加上`top: true`即可。试了一下，能置顶。。只是文章置顶在某一页而不是首页。。

期间看到了`Pacman`主题支持置顶，需要在`_config.yml`中配置好需要置顶的文章，略麻烦。

然后看到了`next`主题支持置顶，在博文`front-matter`中加上`sticky: Sticky`即可置顶，根据`Sticky`的大小来决定置顶顺序。

想实现`next`主题那样的功能，参考了一篇博文[添加Hexo置顶功能的操蛋3小时](http://ziorix.com/2015/09/09/%E6%B7%BB%E5%8A%A0Hexo%E7%BD%AE%E9%A1%B6%E5%8A%9F%E8%83%BD%E7%9A%84%E6%93%8D%E8%9B%8B3%E5%B0%8F%E6%97%B6/)，在`hexo-generator-index`中增加比较函数比较`top`值，我试了一下`Bug`还是有的，置顶文章后文章日期有些会乱掉（比较函数条件比较少）。

我自己写了一个比较函数，也有问题，后来查了一下`Javascript`的`sort`函数，其比较函数和`C++`的完全不同= =

`C++`的比较函数：

```cpp
template<class T>bool cmp(T a, T b) {
	return  a < b; // 升序，降序的话就 b > a
}
```

而`Javascript`的比较函数：

```javascript
cmp(var a, var b) {
	return  a - b; // 升序，降序的话就 b - a
}
```

用了`C++`的比较函数，结果当然会出问题，期间都想重写`js`的排序函数了。。经过修改，已经能完美置顶了，只需要在`front-matter`中设置需要置顶文章的`top`值，将会根据`top`值大小来选择置顶顺序。（大的在前面）

以下是最终的`node_modules/hexo-generator-index/lib/generator.js`

```javascript
'use strict';

var pagination = require('hexo-pagination');

module.exports = function(locals){
  var config = this.config;
  var posts = locals.posts;

	posts.data = posts.data.sort(function(a, b) {
		if(a.top && b.top) { // 两篇文章top都有定义
			if(a.top == b.top) return b.date - a.date; // 若top值一样则按照文章日期降序排
			else return b.top - a.top; // 否则按照top值降序排
		}
		else if(a.top && !b.top) { // 以下是只有一篇文章top有定义，那么将有top的排在前面（这里用异或操作居然不行233）
			return -1;
		}
		else if(!a.top && b.top) {
			return 1;
		}
		else return b.date - a.date; // 都没定义按照文章日期降序排

	});

  var paginationDir = config.pagination_dir || 'page';

  return pagination('', posts, {
    perPage: config.index_generator.per_page,
    layout: ['index', 'archive'],
    format: paginationDir + '/%d/',
    data: {
      __index: true
    }
  });
};
```
