---
title: Linux POSIX信号量解决经典进程同步问题
toc: true
top: false
original: true
date: 2016-09-24 19:28:01
tags:
- Linux
- 操作系统
categories:
- 学习笔记
---

之前我写了一篇博文：[操作系统之进程同步](/2016/09/13/操作系统之进程同步/)，关于多进程（线程）同步的问题，这里我来继续扩展下主题，如何使用信号量来解决进程之间的同步问题，不过我接下来实现还是使用**线程**，理由很简单我之前说过了。

在Unix/Linux上，信号量有2种标准：`System V`和`Posix`。我选择了`Posix`，因为它的接口很简单，功能又和`System V`相近。

不过有些情况可以用**互斥体**来实现，信号量有时候称为并发式编程的`goto`，因为它能在一个进程中唤醒另一个进程，很容易产生出糟糕的同步设计。
<!-- more -->

### 信号量介绍
信号量是同步进程的一个机制，由内核维护的整数（限制>=0），通常用来表示一个共享资源的数量。在一个信号量上可以执行以下操作：
* 将信号量设置成一个初值
* 在信号量上+1(wait原子操作)
* 在信号量上-1(signal/post原子操作)
* 信号量为0

后两种操作可能会导致进程阻塞。第2,3种操作通常称为`P/V`操作。

### Posix信号量接口
使用的`Posix`信号量接口声明在`<semaphore.h>`中。
```cpp
typedef union
{
	char __size[__SIZEOF_SEM_T];
	long int __align;
} sem_t;

/* Initialize semaphore object SEM to VALUE.  If PSHARED then share it
   with other processes.  */
extern int sem_init (sem_t *__sem, int __pshared, unsigned int __value);

/* Wait for SEM being posted. */
extern int sem_wait (sem_t *__sem);

/* Post SEM.  */
extern int sem_post (sem_t *__sem);

/* Get current value of SEM and store it in *SVAL.  */
extern int sem_getvalue (sem_t *__restrict __sem, int *__restrict __sval);

/* Free resources associated with semaphore object SEM.  */
extern int sem_destroy (sem_t *__sem);
```

接口都知道了，开搞。

### 生产者-消费者问题
我们来看看生产者消费者问题。生产者消费者问题是一个著名的线程同步问题，该问题描述如下：有一个生产者在生产产品，这些产品将提供给若干个消费者去消费，为了使生产者和消费者能并发执行，在两者之间设置一个具有多个缓冲区的缓冲池，生产者将它生产的产品放入一个缓冲区中，消费者可以从缓冲区中取走产品进行消费，显然生产者和消费者之间必须保持同步，即不允许消费者到一个空的缓冲区中取产品，也不允许生产者向一个已经放入产品的缓冲区中再次投放产品。

代码很容易写出来：
```cpp
/*************************************************************************
	> File Name: producerConsumer.cpp
	  > Author: Netcan
	  > Blog: http://www.netcan666.com
	  > Mail: 1469709759@qq.com
	  > Created Time: 2016-09-24 六 20:04:04 CST
 ************************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <pthread.h>
#include <semaphore.h>
#define N 10

typedef int item;
// 环形队列
int in = 0, out = 0;
// 缓冲区
item buffer[N];
// 互斥、空缓冲区、满缓冲区信号量
sem_t mutex, empty, full;

// 显示缓冲区内容
void show_buffer() {
//	printf("in=%d, out=%d\n", in, out);
	printf("[");
	if(in == out) // 满了
		for(int i=0; i<N; ++i)
			printf("%2d", buffer[i]);
	else if(out < in)
		for(int i=out; (i%N)<in; ++i)
			printf("%2d", buffer[i]);
	else {
		for(int i=out; i<N; ++i)
			printf("%2d", buffer[i]);
		for(int i=0; i<in; ++i)
			printf("%2d", buffer[i]);
	}
	printf(" ]\n");
}

// 生产者
void* producer(void* id) {
	while(true) {
		item nextp = rand() % 10;
		sem_wait(&empty);
		sem_wait(&mutex);
		buffer[in] = nextp;
		in = (++in)%N;
		printf("======%s======\n", (char*)id);
		printf("生产了：%d\n", nextp);
		show_buffer();
		sem_post(&mutex);
		sem_post(&full);
	}
}

// 消费者
void* consumer(void* id) {
	while(true) {
		sem_wait(&full);
		sem_wait(&mutex);
		item nextc = buffer[out];
		out = (++out)%N;
		int emp, ful;
		sem_getvalue(&empty, &emp);
		sem_getvalue(&full, &ful);
		printf("======%s======\n"
				"缓冲区大小：%d\n"
				"消费的物品：%d\n"
				"空缓冲区数量：%d\n"
				"满缓冲区数量：%d\n"
				,(char*)id, N, nextc, emp, ful);
		show_buffer();
		sem_post(&mutex);
		sem_post(&empty);
	}
}

int main()
{
	// 初始化信号量
	if(sem_init(&mutex, 0, 1) == -1)
		perror("sem_init");
	if(sem_init(&empty, 0, N) == -1)
		perror("sem_init");
	if(sem_init(&full, 0, 0) == -1)
		perror("sem_init");

	// 2个生产者、2个消费者
	pthread_t p1, p2, c1, c2;

	// 并发执行
	if(pthread_create(&p1, NULL, producer, (void*)"生产者1"))
		perror("pthread_create");
	if(pthread_create(&p2, NULL, producer, (void*)"生产者2"))
		perror("pthread_create");
	if(pthread_create(&c1, NULL, consumer, (void*)"消费者1"))
		perror("pthread_create");
	if(pthread_create(&c2, NULL, consumer, (void*)"消费者2"))
		perror("pthread_create");
	if(pthread_join(p1, NULL)) perror("pthread_join");
	if(pthread_join(p2, NULL)) perror("pthread_join");
	if(pthread_join(c1, NULL)) perror("pthread_join");
	if(pthread_join(c2, NULL)) perror("pthread_join");

	return 0;
}
```
首先我们定义一个环形队列表示缓冲区，`in/out`来表示当前生产者、消费者的缓冲区位置，然后定义`empty`和`full`表示空缓冲区和满缓冲区数目，易知一开始`empty = n`，`full = 0`，倘若刚开始消费者抢到cpu，则由于`full`为0导致阻塞，直到生产者生产物品，令`full++`，才能唤醒消费者消费。这里的`mutex`作用是同一时间段只能让一个线程访问缓冲区，即我生产者在生产后将物品放入缓冲区的时候，其他生产者、消费者不能访问缓冲区。

注意`wait`操作的`empty`和`mutex`之间的顺序不能互换，否则会产生死锁，比如我缓冲区满了，`mutex=1`,`empty=0`，这时候生产者再更新缓冲区的时候就会死锁了，消费者的情况同理。

下面来看看程序的执行结果，我只摘抄了具代表性的内容：
```bash
<略>
======生产者2======
生产了：2
[ 2 9 1 2 ]
======消费者1======
缓冲区大小：10
消费的物品：2
空缓冲区数量：5
满缓冲区数量：1
[ 9 1 2 ]
======生产者1======
生产了：6
[ 9 1 2 6 ]
======消费者2======
缓冲区大小：10
消费的物品：9
空缓冲区数量：5
满缓冲区数量：1
[ 1 2 6 ]
======生产者2======
生产了：7
[ 1 2 6 7 ]
======消费者1======
缓冲区大小：10
消费的物品：1
空缓冲区数量：5
满缓冲区数量：1
[ 2 6 7 ]
<略>
```
因为程序的异步的，所以输出结果可能不会是顺序的。

为了便于分析，我只让一个生产者、一个消费者进行实验，得出如下结果：

```bash
<略>
======生产者1======
生产了：2
[ 3 6 7 5 3 5 6 2 ]
======生产者1======
生产了：9
[ 3 6 7 5 3 5 6 2 9 ]
======生产者1======
生产了：1
[ 3 6 7 5 3 5 6 2 9 1 ]
======消费者1======
缓冲区大小：10
消费的物品：3
空缓冲区数量：0
满缓冲区数量：9
[ 6 7 5 3 5 6 2 9 1 ]
======消费者1======
缓冲区大小：10
消费的物品：6
空缓冲区数量：1
满缓冲区数量：8
[ 7 5 3 5 6 2 9 1 ]
======消费者1======
缓冲区大小：10
消费的物品：7
空缓冲区数量：2
满缓冲区数量：7
[ 5 3 5 6 2 9 1 ]
<略>
```

易知，`empty+full == 9`，因为我打印的结果是`wait(full)`后的结果，所以为9不为10。


### 读者-写者问题
多个读者、多个写者共享资源（例如文件、数据等），允许多个读者同时访问资源，写者写时不允许其他进程读或写(互斥访问资源)。
* 读者进程：不控制
* 写者进程：互斥
* 读者写者顺序：无

和上一个问题类似，这里的区别在于，写者与写者或读者互斥，读者之间无限制。

不说了，先写代码。

```cpp
/*************************************************************************
	> File Name: writerReader.cpp
	  > Author: Netcan
	  > Blog: http://www.netcan666.com
	  > Mail: 1469709759@qq.com
	  > Created Time: 2016-09-24 六 21:38:12 CST
 ************************************************************************/
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <pthread.h>
#include <semaphore.h>
#define N 128

int data[N];
int datap = 0;
sem_t rmutex, wmutex;
int readCnt = 0;

// 读者
void *reader(void* id) {
	sem_wait(&rmutex);
	if(readCnt == 0) sem_wait(&wmutex);
	++readCnt;

	// 下面的读操作应该放在rmutex外面，但是为了结果好看，就放在里面了
	printf("======%s======\n", (char*)id);
	for(int i=0; i<datap; ++i)
		printf("%c", data[i]);
	puts("");
	sem_post(&rmutex);

	// 读操作

	sem_wait(&rmutex);
	--readCnt;
	if(readCnt == 0) sem_post(&wmutex);
	sem_post(&rmutex);

	return NULL;
}

// 写者，随意写入一个128字节文本
void *writer(void *id) {
	sem_wait(&wmutex);
	// 写
	if(datap == N) datap = 0;
	printf("======%s======\n", (char*)id);
	for(; datap < N; ++datap)
		data[datap++] = rand()%26 + 'a';
	sem_post(&wmutex);
	return NULL;
}


int main()
{
	// 初始化信号量
	sem_init(&rmutex, 0, 1);
	sem_init(&wmutex, 0, 1);

	// 一个写者，3个读者
	pthread_t w1, r1, r2, r3;
	while(true) {
		if(pthread_create(&w1, NULL, writer, (void*)"写者"))
			perror("pthread_create");
		if(pthread_create(&r1, NULL, reader, (void*)"读者1"))
			perror("pthread_create");
		if(pthread_create(&r2, NULL, reader, (void*)"读者2"))
			perror("pthread_create");
		if(pthread_create(&r3, NULL, reader, (void*)"读者3"))
			perror("pthread_create");
		if(pthread_join(r1, NULL)) perror("pthread_join");
		if(pthread_join(r2, NULL)) perror("pthread_join");
		if(pthread_join(r3, NULL)) perror("pthread_join");
		if(pthread_join(w1, NULL)) perror("pthread_join");
	}

	return 0;
}
```

这里因为可以有任意个读者访问内容，所以当我在读的时候，是不允许你写的；当你在写的时候，是不允许其他线程读、写。

所以我这里用`readCnt`标记正在读的线程数，初始化为0，倘若第一个读者抢到cpu，由于`readCnt==0`，所以将`wait(wmutex)`，后面若写者进行写的时候就会被阻塞。若其他读线程访问内容，也会对`readCnt`做出修改，所以这个`readCnt`为读者线程的临界资源，需要在临界资源前后用`rmutex`互斥访问。当最后一个读者线程结束时，`readCnt==0`，这时候会`signal(wmutex)`，即会唤醒写者线程，后面若有读者要访问，将会被阻塞，直到写线程结束。

运行效果：
```bash
======写者======
======读者3======
rgoqkzrlqiiecwukozljwpeulyharmckvrafsrqibaodyinnjbygsccdbkfuyket
======读者2======
rgoqkzrlqiiecwukozljwpeulyharmckvrafsrqibaodyinnjbygsccdbkfuyket
======读者1======
rgoqkzrlqiiecwukozljwpeulyharmckvrafsrqibaodyinnjbygsccdbkfuyket
======写者======
======读者1======
deavxtfyttcubphnqfvkhxokjvgihkdkqgfnzkmudqohfvuycrimoyyawfkdrpok
======读者2======
deavxtfyttcubphnqfvkhxokjvgihkdkqgfnzkmudqohfvuycrimoyyawfkdrpok
======读者3======
deavxtfyttcubphnqfvkhxokjvgihkdkqgfnzkmudqohfvuycrimoyyawfkdrpok
```
### 参考资料
* 《计算机操作系统（第四版）》
* 《Linux/Unix系统编程手册》第53章


