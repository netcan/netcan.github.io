title: '解决八数码问题之Astar算法'
toc: true
original: true
date: 2016-04-08 08:09:24
tags:
- 优先队列
- 搜索
categories:
- 编程
---

## 扯淡
之前用过宽度搜索写过[八数码问题](/2015/05/28/八数码问题/)，这里用A\*算法实现之。

`Astar`算法居然可以作为毕业论文题材，要是我在计科毕业岂不是轻轻松松的事么。。

## 游戏下载(2016/04/12)
这几天顺便将这个游戏加了个GUI，利用SDL编写的，项目地址：[https://github.com/netcan/SlidePuzzle](https://github.com/netcan/SlidePuzzle)
![https://raw.githubusercontent.com/netcan/SlidePuzzle/master/Slidepuzzle.png](https://github.com/netcan/SlidePuzzle/raw/master/Slidepuzzle.png)

* Windows版本: [slidepuzzle_windows_release_1.1.zip](https://raw.githubusercontent.com/netcan/SlidePuzzle/master/dist/slidepuzzle_windows_release_1.1.zip)
* Linux版本： [slidepuzzle_linux](https://raw.githubusercontent.com/netcan/SlidePuzzle/master/dist/slidepuzzle_linux)

## 何为Astar算法
`Astar`为启发式搜索算法，相对宽度搜索算法，他仅仅多了个评估值$H$，使状态空间搜索不那么盲目。

### $F, G, H$值
给每个状态定义一个$F$值，使$F=G+H$，这个公式的含义：
* $G$表示从初始状态到当前状态转移的量（移动量）
* $H$评估值表示从当前状态到目标状态的转移量评估值（移动量估算值）

### 具体步骤
下面我总结的具体步骤：
1. 新建一个`Open`和`Closed`表，置空，将初始状态放入`Open`表中。
2. 将`Open`表中**最小**$F$值状态取出放入`Closed`表中，若找到目标状态，则中断过程，此时状态的$G$值就是最短路径（在八数码是最少步骤），否则生成其下一步**所有**能转移的状态集$S$
3. 计算状态集$S$每个状态的$G, H$值，得出$F = G+H$值，这里的$G$可取前一个状态（父状态）的$G$值加一，为了寻得最短路径，可将其指向父状态。
4. 枚举状态集，按5, 6处理
5. 若状态集$S$的状态在`Closed`表中，则不理会，否则6
6. 若状态集$S$的状态在`Open`表中且其$F$值较小，则更新（因为有最优的路径啊当然要更新），否则不理会，若不在`Open`表中，则插入。
7. 进入2

### 如何求得路径
前面提到过，可将转移的状态指向其父状态，那么最后只需要在`Closed`中从**目标**状态往回找**初始**状态即可。

## 何为八数码
在3×3的棋盘上，摆有八个棋子，每个棋子上标有1至8的某一数字。棋盘中留有一个空格，空格用0来表示。空格周围的棋子可以移到空格中。求出从初始状态到目标状态的最少步骤和路径（解法）。

### 使用Astar需要注意的细节

#### 优先队列的实现
由于`Open`表是个优先队列（最小堆），刚开始我很本能得用最小堆`priority_queue`来搞，到后面发现优先队列是不能修改/查找元素的= =后来改成每次用`vector`+排序+查找搞定，虽然效率低但是考虑到八数码共$9! = 362880$种状态，还能接受。

当然这里有需要注意的地方，我昨晚跑了几分钟没出结果，去洗了个澡，神清气爽恍然大悟，问题出在$H$值的评估。

#### $H$的单调性
这里在强调一下，$F = G + H$，通俗来讲$G$值表示**初始**状态到**当前**状态的移动量，每转移一个状态，移动量$G$就加1。$H$表示从**当前**状态到**目标**状态的评估值。易知，我们在搜索**目标**状态时，$G$会越来越大（因为离**初始**状态越来越远），$H$会越来越小（因为离**目标**状态越来越近），最终找到**目标**状态时，应该有$F = G + H = G + 0$，所以我前面说，最后的$G$值就是最短路径（最少步骤）了。

那么我昨晚找不到结果的问题，出在$H$的单调性。。。随着状态的转移深入，我的$H$还越来越大，后来将$H$值改成递减函数，就能出结果了= =

我好像忘记说了$H$怎么求得的，这里因为是评估函数，只要保证离**目标**状态越近，它越小就好了，联系**当前**状态和**目标**状态，可以这样求得：**当前**状态与**目标**状态的3x3矩阵中不同数字的个数即为$H$，例如：

当前状态：

4|6|3
:-:|:-:|:-:
2|8|5
1|0|7

目标状态：

1|2|3
:-:|:-:|:-:
4|5|6
7|8|0

$H=8$，两个状态只有$(1, 3)$的数是相等的，所以有$H=8$个数是不等的。显然$H=0$的时候两个状态相同。

#### 数据结构的选择(2016/04/11更新)
有组数据

	起始: 014276385
	终止: 123456780

跑了将近5分钟。正好上课迟到了那就回来优化下数据结构，将前面用`vector`遍历+排序的算法优化成现在的双`map`*键-值对*与*值-键*对，因为map的key就是**有序**的，并利用*序列化*+*二分搜索*的方法，时间优化到10ms，太爽了= =

#### 判断是否有解
可是你以为调个$H$的单调性就能解决问题么？Naive!

如果无解的话这货会一直搜索下去，我怀疑有圈= =

所以我又找到了一个判断是否有解的算法，根据八数码的奇偶序列来判断，如果两个八数码奇偶序列相同（同奇同偶）则有解。

> 一个状态表示成一维的形式，求出除0之外所有数字的逆序数之和，也就是每个数字前面比它大的数字的个数的和，称为这个状态的逆序。

> 若两个状态的逆序奇偶性相同，则可相互到达，否则不可相互到达。

奇偶序列的计算（举个例子）：

4|6|3
:-:|:-:|:-:
2|8|5
1|0|7

写成一维就是`463285107`，奇偶序列为$\color{red}{0}(4)+\color{red}{1}(6)+ \color{red}{0}(3)+ \color{red}{0}(2) +\color{red}{4}(8)+\color{red}{3}(5)+0(1)+\color{red}{0}(0)+\color{red}{6}(7) = \color{red}{14}$

### 实现代码1(`Vector`枚举+排序)
```cpp
/*************************************************************************
	> File Name: eight.cpp
	  > Author: Netcan
	  > Blog: http://www.netcan666.com
	  > Mail: 1469709759@qq.com
	  > Created Time: 2016-04-07 四 18:24:18 CST
 ************************************************************************/

#include <iostream> // C++的IO输入输出
#include <cstdio> // C的输入输出
#include <vector> // 存储open表的数据结构
#include <map> // 存储closed表的数据结构
#include <cstring> // 含memcpy的接口
#include <algorithm> // 排序
using namespace std;

typedef int board[4][4]; // 二维数组，下标从1开始，用于存放八数码的状态
int movx[] = {-1, 0, 1, 0}; // 空格下右上左四个方向移动
int movy[] = {0, 1, 0, -1};
bool success(const board &cur, const board &e); // 判断两个状态是否相等

struct Status {
	board s; // 利用二维数组存储当前状态
	int f, g, h; // f = g + h
	int cid, pid; // 当前状态id，其父节点id
	Status(const board &s, int f, int g, int h, int cid, int pid) { // 构造函数
		memcpy(this->s, s, sizeof(board)); // 复制状态到s
		this->f = f;
		this->g = g;
		this->h = h;
		this->cid = cid;
		this->pid = pid;
	}
	bool operator<(const Status &b) const { // 重载<，排序函数sort需要
		return f < b.f;
	}
	bool operator==(const Status &b) const { // 重载==，find查找元素需要
		return success(s, b.s);
	}
};

vector<Status> opening; // 优先队列（vector排序）存储当前待搜索的状态
map<int, bool> closed; // 记录使用过的状态
vector<Status> path; // 保存路径，即closed表的详尽结构

void input(board s) { // 输入状态
	for(int i=1; i<=3; ++i)
		for(int j=1; j<=3; ++j)
			scanf("%1d", &s[i][j]);
}
void output(const board &s) { // 输出状态
	for(int i=1; i<=3; ++i) {
		for(int j=1; j<=3; ++j)
			printf("%2d", s[i][j]);
		puts("");
	}
}

int H(const board &cur,const board &e) { // 评估函数，当H==0的时候说明状态相等，H需要收敛
	int h = 0;
	for(int i=1; i<=3; ++i)
		for(int j=1; j<=3; ++j)
			if(cur[i][j] != e[i][j]) ++h;
	return h;
}

int F(const Status &s) { // F = G + H
	return s.g + s.h;
}

bool success(const board &cur, const board &e) { // 判断是否到达终点
	return H(cur, e) == 0;
}

int serialization(const board &cur) { // 状态序列化成一维数字，方便存储。
	int sr = 0;
	for(int i=1; i<=3; ++i)
		for(int j=1; j<=3; ++j)
			sr = sr * 10 + cur[i][j];
	return sr; // 序列化结果
}

bool checkvalid(const board &s, const board &e) { // 判断是否有解
	int ss = 0, ee = 0;
	for(int i=0; i<9; ++i)
		for(int j=0; j<i; ++j) {
			if(s[j/3+1][j%3+1] != 0 && s[j/3+1][j%3+1] < s[i/3+1][i%3+1]) ++ss;
			if(e[j/3+1][j%3+1] != 0 && e[j/3+1][j%3+1] < e[i/3+1][i%3+1]) ++ee;
		}
	return (ss&1) == (ee&1); // 同奇同偶有解
}

int run(const board &s,const board &e) { // Astar
	if(!checkvalid(s, e)) return -1; // 判断是否有解，无解返回-1
	// 清理Open/Close/Path表
	opening.clear();
	closed.clear();
	path.clear();
	int id = 0; // 每使用一个id，就++，随着状态转移，id只会越来越大
	opening.push_back(Status(s, H(s,e), 0, H(s, e), id, id++)); // 初始状态放入closed表中
	while(opening.size()) { // 直到表空，如果无解的话是不可能表空的= =
		sort(opening.begin(), opening.end()); // 排序求得最小F值
		Status p = *opening.begin(); // 取出最小F值的状态
		opening.erase(opening.begin()); 　// 删除最小F值的状态
		path.push_back(p); // 放到path数据结构中，用来保存路径
		closed[serialization(p.s)] = true; // 将状态s放入closed表中
		if(success(p.s, e)) return p.g; // 找到目标状态，返回最少步骤
		int cx = -1, cy = -1; // 寻找空格的位置（即0）
		for(int i=1; i<=3; ++i) {
			if(cx != -1) break;
			for(int j=1; j<=3; ++j)
				if(p.s[i][j] == 0) { // 找到空格
					cx = i;
					cy = j;
					break;
				}
		}
		for(int k=0; k<4; ++k) { // 生成4个移动到相邻格子的状态（相当于空格上下左右移动）
			int nx = cx + movx[k];
			int ny = cy + movy[k];
			if(nx >= 1 && nx <= 3 && ny >= 1 && ny <= 3) { // 移动判断是否越界
				swap(p.s[cx][cy], p.s[nx][ny]); // 开始移动
				Status n(p.s, 0, p.g+1, H(p.s, e), id++, p.cid); // 生成的状态
				swap(p.s[cx][cy], p.s[nx][ny]); // 恢复原位
				n.f = F(n); // f=g+h
				// 查找Open表中是否有这个状态，没有就添加，有的话更新最小值
				if(!closed.count(serialization(n.s))) { // 先查找是否在closed表中
					vector<Status>::iterator it = find(opening.begin(), opening.end(), n); // 遍历查找状态
					if(it != opening.end()) { // 找到
						if(it->f > n.f) { // 有最优解
							opening.erase(it); // 删除
							opening.push_back(n); // 更新
						}
					}
					else
						opening.push_back(n); // 添加
				}
			}
		}
	}
	return -1; // 无解= =
}

void outpath(int pid, int ss, int step) { // 后序递归寻找路径
	if(step < 0) return; // 无解
	else if(step == 0) { // 初始状态
		printf("Step(%d) ==> \n", step);
		output(path[ss].s); // 输出状态
		return;
	}
	for(int i=ss; i>=0; --i) // 往前查找父状态
		if(path[i].cid == pid)  // 找到父状态
			outpath(path[i].pid, i, step - 1); // 后序递归
	printf("Step(%d) ==> \n", step);
	output(path[ss].s); // 输出状态
}

int main() {
	freopen("eight.in", "r", stdin); // 从文件中读取数据
	board start, end;
	cout << "输入起始状态(9个数，0表示空格）：" << endl;
	input(start);
	cout << "输入目标状态(9个数，0表示空格）：" << endl;
	input(end);
	int step = run(start, end); // Astar
	if(step != -1)
		outpath((path.end()-1)->pid, path.size() - 1, step); // 输出路径
	else
		cout << "无解！" << endl;

	return 0;
}
```

### 实现代码2(双`map`+二分)
```cpp
/*************************************************************************
    > File Name: eight.cpp
      > Author: Netcan
      > Blog: http://www.netcan666.com
      > Mail: 1469709759@qq.com
      > Created Time: 2016-04-07 四 18:24:18 CST
 ************************************************************************/

#include <iostream> // C++的IO输入输出
#include <cstdio> // C的输入输出
#include <vector> // 存储open表的数据结构
#include <map> // 存储closed表的数据结构
#include <cstring> // 含memcpy的接口
#include <algorithm> // 排序
using namespace std;

typedef int board[4][4]; // 二维数组，下标从1开始，用于存放八数码的状态
int movx[] = {-1, 0, 1, 0}; // 空格下右上左四个方向移动
int movy[] = {0, 1, 0, -1};
bool success(const int &s, const int &e); // 判断两个状态是否相等

struct Status {
    int s; // 利用一维数据存储当前状态
    int g, h; // f = g + h
    int cid, pid; // 当前状态id，其父节点id
    Status(const int s, int g, int h, int cid, int pid): s(s), g(g), h(h), cid(cid), pid(pid) {}
    bool operator<(const Status &b) const { // 根据状态排序
        return s < b.s;
    }
    bool operator==(const Status &b) const { // 重载==，find查找元素需要
        return success(s, b.s);
    }
};

// 将opening表分成两部分来维护，提高效率
multimap<int, Status> opening_key; // F-value
map<Status, int> opening_value; // value-F

map<int, bool> closed; // 记录使用过的状态
vector<Status> path; // 保存路径，即closed表的详尽结构

void ser2board(int s, board t);

void input(board s) { // 输入状态
    for(int i=1; i<=3; ++i)
        for(int j=1; j<=3; ++j)
            scanf("%1d", &s[i][j]);
}
void output(const int &s) { // 输出状态
	board tmp;
	ser2board(s, tmp);
    for(int i=1; i<=3; ++i) {
        for(int j=1; j<=3; ++j)
            printf("%2d", tmp[i][j]);
        puts("");
    }
}

int H(int cur, int e) { // 评估函数，当H==0的时候说明状态相等，H需要收敛
	int d = 0;
	for(int i=0; i<9; ++i) {
		if(cur % 10 != e % 10) ++d;
		cur/=10;
		e/=10;
	}
	return d;
}

int F(const Status &s) { // F = G + H
    return s.g + s.h;
}

bool success(const int &cur, const int &e) { // 判断是否到达终点
    return H(cur, e) == 0;
}

int serialization(const board &cur) { // 状态序列化成一维数字，方便存储。
    int sr = 0;
    for(int i=1; i<=3; ++i)
        for(int j=1; j<=3; ++j)
            sr = sr * 10 + cur[i][j];
    return sr; // 序列化结果
}

void ser2board(int s, board t) {
	// printf("%d\n", s);
	for(int i=3; i>=1; --i)
		for(int j=3; j>=1; --j) {
			t[i][j] = s%10;
			// printf("%d\n", t[i][j]);
			s /= 10;
		}
}

bool checkvalid(const int &s, const int &e) { // 判断是否有解
    int ss = 0, ee = 0;
	board tmps, tmpe;
	ser2board(s, tmps);
	ser2board(e, tmpe);
    for(int i=0; i<9; ++i)
        for(int j=0; j<i; ++j) {
            if(tmps[j/3+1][j%3+1] != 0 && tmps[j/3+1][j%3+1] < tmps[i/3+1][i%3+1]) ++ss;
            if(tmpe[j/3+1][j%3+1] != 0 && tmpe[j/3+1][j%3+1] < tmpe[i/3+1][i%3+1]) ++ee;
        }
    return (ss&1) == (ee&1); // 同奇同偶有解
}


int run(const int &s,const int &e) { // Astar
	if(!checkvalid(s, e)) return -1; // 判断是否有解，无解返回-1
	// 清理Open/Close/Path表
	opening_key.clear();
	opening_value.clear();
	closed.clear();
	path.clear();
	int id = 0; // 每使用一个id，就++，随着状态转移，id只会越来越大
	Status st(s, 0, H(s, e), id, id++);
	opening_key.insert(make_pair<int, Status>(H(s, e), st)); // 初始状态放入closed表中
	opening_value.insert(make_pair<Status, int>(st, H(s, e))); // 初始状态放入closed表中

	while(opening_key.size()) { // 直到表空，如果无解的话是不可能表空的= =
		Status p = opening_key.begin()->second; // 取出最小F值的状态

		opening_key.erase(opening_key.begin()); // 删除最小F值的状态
		opening_value.erase(opening_value.lower_bound(p));

		path.push_back(p); // 放到path数据结构中，用来保存路径
		closed[p.s] = true; // 将状态s放入closed表中
		if(success(p.s, e)) return p.g; // 找到目标状态，返回最少步骤
		int cx = -1, cy = -1; // 寻找空格的位置（即0）
		board tmp;
		ser2board(p.s, tmp);
		for(int i=1; i<=3; ++i) {
			if(cx != -1) break;
			for(int j=1; j<=3; ++j)
				if(tmp[i][j] == 0) { // 找到空格
					cx = i;
					cy = j;
					break;
				}
		}
		for(int k=0; k<4; ++k) { // 生成4个移动到相邻格子的状态（相当于空格上下左右移动）
			int nx = cx + movx[k];
			int ny = cy + movy[k];
			if(nx >= 1 && nx <= 3 && ny >= 1 && ny <= 3) { // 移动判断是否越界
				swap(tmp[cx][cy], tmp[nx][ny]); // 开始移动
				int tmps = serialization(tmp);
				Status n(tmps, p.g+1, H(tmps, e), id++, p.cid); // 生成的状态
				swap(tmp[cx][cy], tmp[nx][ny]); // 恢复原位
				int f = F(n); // f=g+h
				// 查找Open表中是否有这个状态，没有就添加，有的话更新最小值
				if(!closed.count(tmps)) { // 先查找是否在closed表中
					// vector<Status>::iterator it = find(opening.begin(), opening.end(), n); // 遍历查找状态
					map<Status, int>::iterator itv = opening_value.lower_bound(n);
					map<int, Status>::iterator itk;
					if(itv != opening_value.end() && itv->first == n) { // 找到
						if(F(itv->first) > f) { // 有最优解
							for(itk = opening_key.lower_bound(F(itv->first)); itk != opening_key.upper_bound(F(itv->first)); ++itk)
								if(itk->second == n) break;
							opening_value.erase(itv); // 删除
							opening_key.erase(itk);
							opening_key.insert(make_pair<int, Status>(f, n)); // 初始状态放入closed表中
							opening_value.insert(make_pair<Status, int>(n, f)); // 初始状态放入closed表中
						}
					}
					else {
						opening_key.insert(make_pair<int, Status>(f, n)); // 初始状态放入closed表中
						opening_value.insert(make_pair<Status, int>(n, f)); // 初始状态放入closed表中
					}
				}
			}
		}
	}
	return -1; // 无解= =
}

void outpath(int pid, int ss, int step) { // 后序递归寻找路径
	if(step < 0) return; // 无解
	else if(step == 0) { // 初始状态
		printf("Step(%d) ==> \n", step);
		output(path[ss].s); // 输出状态
		return;
	}
	for(int i=ss; i>=0; --i) // 往前查找父状态
		if(path[i].cid == pid)  // 找到父状态
			outpath(path[i].pid, i, step - 1); // 后序递归
	printf("Step(%d) ==> \n", step);
	output(path[ss].s); // 输出状态
}

int main() {
	freopen("eight.in", "r", stdin); // 从文件中读取数据
    board start, end;
    cout << "输入起始状态(9个数，0表示空格）：" << endl;
    input(start);
    cout << "输入目标状态(9个数，0表示空格）：" << endl;
    input(end);

	int step = run(serialization(start), serialization(end)); // Astar
	cout << step << endl;
	if(step != -1)
		outpath((path.end()-1)->pid, path.size() - 1, step); // 输出路径
	else
		cout << "无解！" << endl;

    return 0;
}
```

### 运行结果
	输入起始状态(9个数，0表示空格）：
	283104765
	输入目标状态(9个数，0表示空格）：
	123804765
	Step(0) ==>
	2 8 3
	1 0 4
	7 6 5
	Step(1) ==>
	2 0 3
	1 8 4
	7 6 5
	Step(2) ==>
	0 2 3
	1 8 4
	7 6 5
	Step(3) ==>
	1 2 3
	0 8 4
	7 6 5
	Step(4) ==>
	1 2 3
	8 0 4
	7 6 5

	输入起始状态(9个数，0表示空格）：
	014276385
	输入目标状态(9个数，0表示空格）：
	123456780
	Step(0) ==>
	0 1 4
	2 7 6
	3 8 5
	Step(1) ==>
	2 1 4
	0 7 6
	3 8 5
	Step(2) ==>
	2 1 4
	7 0 6
	3 8 5
	Step(3) ==>
	2 0 4
	7 1 6
	3 8 5
	Step(4) ==>
	2 4 0
	7 1 6
	3 8 5
	Step(5) ==>
	2 4 6
	7 1 0
	3 8 5
	Step(6) ==>
	2 4 6
	7 1 5
	3 8 0
	Step(7) ==>
	2 4 6
	7 1 5
	3 0 8
	Step(8) ==>
	2 4 6
	7 1 5
	0 3 8
	Step(9) ==>
	2 4 6
	0 1 5
	7 3 8
	Step(10) ==>
	2 4 6
	1 0 5
	7 3 8
	Step(11) ==>
	2 0 6
	1 4 5
	7 3 8
	Step(12) ==>
	0 2 6
	1 4 5
	7 3 8
	Step(13) ==>
	1 2 6
	0 4 5
	7 3 8
	Step(14) ==>
	1 2 6
	4 0 5
	7 3 8
	Step(15) ==>
	1 2 6
	4 5 0
	7 3 8
	Step(16) ==>
	1 2 0
	4 5 6
	7 3 8
	Step(17) ==>
	1 0 2
	4 5 6
	7 3 8
	Step(18) ==>
	1 5 2
	4 0 6
	7 3 8
	Step(19) ==>
	1 5 2
	4 3 6
	7 0 8
	Step(20) ==>
	1 5 2
	4 3 6
	7 8 0
	Step(21) ==>
	1 5 2
	4 3 0
	7 8 6
	Step(22) ==>
	1 5 2
	4 0 3
	7 8 6
	Step(23) ==>
	1 0 2
	4 5 3
	7 8 6
	Step(24) ==>
	1 2 0
	4 5 3
	7 8 6
	Step(25) ==>
	1 2 3
	4 5 0
	7 8 6
	Step(26) ==>
	1 2 3
	4 5 6
	7 8 0

## 参考链接
* [A星寻路算法介绍](http://www.cnblogs.com/zhoug2020/p/3468167.html)
* [八数码问题有解的条件及其推广](http://blog.sina.com.cn/s/blog_82a8cba50100ur8n.html)




