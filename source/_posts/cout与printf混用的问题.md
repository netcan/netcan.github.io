title: cout与printf混用的问题
toc: false
original: true
date: 2015-05-08 13:17:52
updated: 2015-05-08 13:17:52
tags:
- C/C++
categories:
- 随笔
---

前几天做一道题目的时候因为混用了iostream和stdio导致一直WA而查不出来。。

今天看了一下课件发现cout与printf混用会导致输出顺序问题。。

下面做个实验。。

	int main()
	{
		cout << "Net";
		printf("Can\n");
		return 0;
	}
输出

	NetCan

<!-- more -->
没问题。我上次为了加快cin/cout速度，关闭了同步stdio导致出错，现在试试

	int main()
	{
		ios::sync_with_stdio(false); // 关闭同步
		cout << "Net";
		printf("Can\n");
		return 0;
	}
输出

	Can
	Net
顺序发生了变化。。

sync_with_stdio()是在<ios_base>中定义的，当其接受true作为参数时，将会同步iostream与stdio中的流操作。默认是true，因此第一个程序的结果是正确的。然而，尽管C++标准中规定stdio sync标志默认是true，不同平台下的不同编译器可能并不完全支持这个标准。

下面改一下，

	int main()
	{
		ios::sync_with_stdio(false);
		cout << "Net" << flush;
		printf("Can\n");
		return 0;
	}
输出结果正确。

因为flush强制清空了缓冲区，将其中的内容输出。
