title: 蒙特卡罗法之随机素数判定算法Miller_Rabin与Pollard_Rho质因数分解算法
toc: true
original: true
tags:
  - 数学
  - 数论
categories:
  - 学习笔记
date: 2015-12-06 14:50:53
---
昨天刷到[POJ 2429](http://poj.org/problem?id=2429)，没刷出来= =，难度太大，翻*Discuss*翻到了一题[POJ 1811](http://poj.org/problem?id=1811)，了解了两个算法：`Miller_Rabin`$O(log n)$素数判定与`Pollard_Rho`$O(\sqrt[4] n)$分解质因数算法。

<!--~~看室友打垃圾游戏`LOL`毫不考虑他人的份上，若能争取换宿舍，迟早会一个人住的，远离室友保平安，废物一个。~~-->

## Miller_Rabin算法
这是我从**《算法导论》**和Google搜刮的资料上面整理出来的笔记，若有错误，请指出，谢谢。

### 素数定理：
素数分布函数$\pi(n)$描述了小于等于n的素数数量。
{% raw %}
$$\lim_{n\to \infty} \dfrac{\pi (n)}{n/ln n} = 1$$
{% endraw %}

在$n$非常小的情况下有$\pi(n)\sim n/ln n$，例如$n=10^9$次方时，$\pi(n)=508 475 34$，而$n/ln n \approx 48254942$，误差不超过$6\%$。

随机选取一个数字$n$，通过素数定理，该数字$n$为素数的概率为$1/ln n$，期望值近似为$ln n$，因此为了找出一个长度与$n$相同的素数，需要在n附近随机取$ln n$个整数。


### 试除法
假定$n$具有下列素数分解的因子：
{% raw %}
$$n = p_1^{e_1}p_2^{e_2}\cdots p_r^{e_r}$$
{% endraw %}

当且仅当$r=1$并且$e_1=1$时，$n$是素数。

所以可以用$2, 3, \cdots, [\sqrt n]$去试除，最坏情况下，复杂度达到$\Theta(\sqrt n)$，若$n$非常大显然效率非常低，于是出现了一种高效判定大素数算法。。。

### 费马定理
若$n$是素数，则对任意{% raw %}$a \in \{1, 2, \cdots, n-1\}${% endraw %}，有
{% raw %}
$$ \tag{费马定理}\label{Fermat}a^{n-1} \equiv 1\pmod n$$
{% endraw %}

证：

先引出一个结论：对任意$a<n$的数$a, 2a, 3a, \cdots, (n-1)a$除以$n$的余数正好为1到$n-1$的一个排列，例如5是素数，取$2, 4, 6, 8$，余数为$2, 4, 1, 3$，为1到4的一个排列。

反证法：若结论不成立的话，也就是至少有两个数$i, j < n, i < j$，使得$i a$与$j a$的余数相同，即
{% raw %}
$$ i a = k_1 n+m, j a = k_2 n + m，做差得(j-i)a = (k_2 - k_1)n$$
{% endraw %}

也就是说$n$可以整除$(j-i)a$，而$n$是素数且$j-i \leq n$，与条件矛盾。

那么有
{% raw %}
$$ (n-1)! \equiv a 2a 3a \cdots (n-1)a = (n-1)! a^{n-1}\pmod n即\\
a^{n-1} \equiv 1\pmod n$$
{% endraw %}

### 伪素数测试过程
根据费马定理，称$n$是一个基为$a$的伪素数。

费马定理意味着若$n$是素数，则对$a\leq n$都满足$\eqref{Fermat}$，令人惊讶的是它的逆命题也几乎成立（注意是几乎成立。。）

于是对$a=2$，看看$n$是否满足$\eqref{Fermat}$，若不满足则返回*COMPOSITE*声明$n$是合数，否则返回*PRIME*，猜测$n$是素数（实际是只是$n$或者是素数，或者是基于2的伪素数）

```
if(mod_pow(a, n-1, n) != 1) return COMPOSITE; // 总是正确的
else return PRIME; // 可能错误，如果基于a的伪素数
```

这个过程可能产生错误，但是它判定合数总是正确的。如果判定素数，只有基于2的伪素数时才可能出错。

出错的几率非常小，但是不能无视它，在小于10000的$n$值中，有22个值会产生错误。可惜我们也不能通过取其他基数（例如$a=3$）来消除所有的错误，因为还有一种称作`Carmichael`的**合数**，对任意$a\leq n$，满足$\eqref{Fermat}$。（真抓狂233）

下一步是对素数测试方法改进，使得测试过程中不会将`Carmichael`当成素数。

### Miller_Rabin素数测试方法
Miller_Rabin测试方法对上述测试过程做了改进，克服其存在的问题。

测试方法引进`witness`，当且仅当$a$为合数$n$的证据时，返回`TRUE`，`witness`取$a=2$测试的扩展：
$$a^{n-1} \not \equiv 1\pmod n$$

来介绍下`witness`的构造过程，令$n-1=2^t u$，其中$t\geq 1$且$u$是奇数，即$n-1$的二进制表示是奇数$u$的二进制后面接$t$个$0$。

则$a^{n-1} \equiv (a^u)^{2^t}$，所以可以通过先计算$a^u\ \bmod n$，然后**连续平方**$t$次来计算最终的$a^{n-1}\ \bmod n$，期间若有一次$\neq 1或者\neq n-1，即x_i\equiv (a^u)^{2^i}\pmod n\neq \pm 1, (i=0, 1,\cdots, t)$则返回合数。

现在再来证明一下为什么$x_i\not \equiv \pm 1$时，$n$为合数。

首先引出定理：

如果$p$是一个奇素数($>2$)且$e\geq 1$，则方程
$$x^2\equiv 1\pmod {p^e}$$

仅有两个解，即$x=\pm 1$。

证：

方程等价于(`|`是整除的意思。。)：
$$p^e|x^2-1 \Rightarrow p^e|(x+1)(x-1)$$

这说明了$p|(x-1)$或者$p|(x+1)$，但不同时成立，否则$p$也能整除它们的差$(x+1)-(x-1)=2$。

若$p\not |(x-1)$，则$p^e|(x+1)$，也就是$x\equiv -1\pmod {p^e}$。对称地，得出$x\equiv 1\pmod{p^e}$，综上可得$x\equiv \pm1\pmod{p^e}$。

所以**不断平方**，期间$\not \equiv \pm 1$，则可说明$n$是合数。

下面是`witness`实现过程

	bool witness(ll a, ll n, ll u, ll t) { // a随机数，u, t于外部计算
		ll ret = mod_pow(a, u, n);
		ll last = ret;
		for(int i=1; i<=t; ++i) {
			ret = mod_mult(ret, ret, n);
			if(ret == 1 && last != 1 && last != n-1) return true;
			last = ret;
		}
		if(ret!=1) return true; // a^(n-1)%n!=1
		return false;
	}

举个例子，比如`Carmichael`数$n=561$，首先$n-1=561-1=2^t u=2^4\times 35$，随机取$a=5\leq  561$，这里采用从后往前的顺序计算。

1. 先计算{% raw %}$x_t=a^{n-1}\pmod n${% endraw %}，即{% raw %}$x_t=5^{560}\equiv 1\pmod {561}${% endraw %}
2. 再计算{% raw %}$x_{t-1}=5^{280}\equiv 67\pmod {561}${% endraw %}，直接说明了$561$是合数= =

若按程序从前往后算，{% raw %}$x_0=a^u\pmod n=5^{35}\equiv 23\pmod {561}${% endraw %}，{% raw %}$x_1=(a^u)^2\pmod n=5^{70}\equiv 529\pmod {561}${% endraw %}，直到算到{% raw %}$x_{t-1}, x_t${% endraw %}即能检验出来。

整个$X$序列如下：
{% raw %}
$$ x=<23, 529, 463, 67, 1>$$
{% endraw %}

最后的`Miller_Rabin`函数就出炉了：

	bool Miller_Rabin(ll n, int s) { // 随机取s次a
		if(n==2) return true;
		if(n < 2 || !(n&1)) return false;
		ll u=n-1;
		ll t=0;
		while(!(u&1)) {u>>=1; ++t;} // 2^t*u = n-1
		for(int i=0; i<s; ++i) {
			ll a=rand()%(n-1)+1; // 若n为合数，证据为a的概率至少为1/2
			if(witness(a, n, u, t)) return false; // 合数
		}
		return true;
	}

### Miller_Rabin素数测试的出错率
如果`Miller_Rabin`返回`true`(即`PRIME`)，则它还是有一种很小的可能性出错。然而概率非常低，仅取决于`s`的大小和选取`a`值的运气。。而且每次都需要经过`witness`的严格判断，出错率应该极小，出错概率最多为$(1/2)^s$，然而对于一个长度为`1024`位的数字，大概需要$s=9$次，对于64位数，长度不超过20，计算得$s=1.14$，取$s=9$出错率可忽略不计。

## Pollard_Rho分解质因数算法
~~实验报告抄死人，实验累死人，退化学保平安，待续。。。~~

首先用`Miller_Rabin`判断一个数是否是质数，若不是质数则对其分解质因数。

### 试除法分解合数
试除法前面有介绍，效率较低，复杂度达$\Theta(\sqrt n)$。

试除法是一个一个试，可对其进行随机选取的方法提高效率。

简单起见，假设$n$是一个合数，并且只有两个真因子$p,q(p\times q=n)$。

若从小于$n$的数随机取一个，取到真因子的概率为$2/n$，若$n$比较大，效率也比较低。

### 进一步优化
假设从1000中随机去一个数，为6的概率为$1/1000=0.001$。

若换种方法，每次取两个数$i, j(i\leq j)$，问$j-i$是否为6，概率是{% raw %}$(1000-6)*2/(1000*1000)\approx0.002${% endraw %}，概率提高了。。

试着将这个方法推广：随机选择k个数，并在其中两两比较。（对于n，在$k = \sqrt n$处有一半的命中率。）

所以直接推广，选出$\sqrt n$个数并在之中两两比较，寻找到$n|j - i$（即$i - j = p 或 q$）

可以问题没有这么简单，现在我们要储存$\sqrt n$个数，还要进行$\sqrt n \times \sqrt n$次比较，而且只有一半的概率找到真因子。

### 伪随机函数
显然，我们不能储存$\sqrt n$个数，也不能进行n次比较。

如果我们只是在生成随机数时，比较生成时间上相邻的两个随机数，就可以同时解决上面的两个问题。

有伪随机函数$f(x) = (x ^ 2 + a) \bmod N$，其中N是常数，a是随机生成的常数。 并给出序列{% raw %}$x_1 = c, x_2 = f(x_1), x_3 = f(x_2) \cdots${% endraw %}。（[原文](https://www.cs.colorado.edu/~srirams/courses/csci2824-spr14/pollardsRho.html)没有解释为什么这个函数可以替换上述机制）

### GCD
同时我们可以优化检测手段：我们不检测$n|j-i$，而是检测$p,q=GCD(j - i, n) > 1$

因为只有$p,q$整除$n$，令$x=j-i$，则
{% raw %}
$$
x = p, = 2p, = 3p \cdots = (q - 1) \times p\\
x = q, = 2q, = 3q \cdots = (p - 1) \times q\\
(p \times q不再内，因为i \leq j < n)$$
{% endraw %}

共有$p+q-2$个。

### 判环
看似大功告成了，不过我们引入的伪随机函数$f(x)$还有问题，$f(x)$生成的数的轨迹有可能有环（这也是算法名字中Rho($\rho$)的来历）。也就是对$f(x)$的不断迭代可能陷入一个死循环。

为此我们需要在终点未知而空间有限的条件下判断是否陷入环中。

如果发现了有环，可以重新（随机）给出常数$x_1 = c$，再次运行算法。

### 最终代码

	ll Pollard_rho(ll n, ll c) { // 伪随机函数f(x)=x^2+c，c为随机数
		ll i=1, k=2;
		ll x=rand()%n;
		ll y = x;
		while(1) {
			++i;
			x=(mod_mult(x, x, n)+c)%n; // 伪随机数
			ll d = gcd(y-x+n, n); // gcd注意负数
			if(d!=1 && d!=n) return d;
			if(y==x) return n; // 遇环退出
			if(i==k) {y=x; k<<=1;}
		}
	}

### 递归分解并储存所有质因数
这里再给出递归分解所有质因数的方法。

	vector<ll> factors; // 储存质因数分解结果（无序）
	void findfac(ll n) {
		if(Miller_Rabin(n, 20)) { // 素数
			factors.push_back(n); // 储存素因子
			return;
		}
		ll p = n;
		while(p>=n) p = Pollard_rho(p, rand()%(n-1)+1); // 找出当前合数的一个素因子
		findfac(p);
		findfac(n/p);
	}


## 习题

### POJ 1811
题目地址：[http://poj.org/problem?id=1811](http://poj.org/problem?id=1811)

#### 题目大意
给你一个64位大整数，判断是否为素数，若不是，则找出最小质因子。

#### 思路
直接用`Miller_Rabin`进行判定，然后`findfac`求出所有质因子，最后输出最小的质因子。

#### AC代码
```cpp
/*************************************************************************
	> File Name: 1811.cpp
	  > Author: Netcan
	  > Blog: http://www.netcan.xyz
	  > Mail: 1469709759@qq.com
	  > Created Time: 2015-12-05 六 14:27:28 CST
 ************************************************************************/

#include <iostream>
#include <vector>
#include <string>
#include <queue>
#include <algorithm>
#include <cmath>
#include <ctime>
#include <cstdio>
#include <sstream>
#include <deque>
#include <functional>
#include <iterator>
#include <list>
#include <map>
#include <memory>
#include <stack>
#include <set>
#include <numeric>
#include <utility>
#include <cstring>
using namespace std;
typedef long long ll;

ll mod_mult(ll a, ll b, ll mod) { //return a*b%mod
	ll s = 0;
	while(b) {
		if(b&1) s=(s+a)%mod;
		a<<=1, b>>=1;
		if(a > mod) a %= mod;
	}
	return s;
}

ll mod_pow(ll x, ll n, ll mod) {  // a^b%mod
	ll res = 1;
	while(n) {
		if(n&1) res = mod_mult(res, x, mod);
		x=mod_mult(x, x, mod);
		n>>=1;
	}
	return res;
}

//以a为基,n-1=u*2^t      a^(n-1)=1(mod n)  验证n是不是合数
//一定是合数返回true,不一定返回false
bool witness(ll a, ll n, ll u, ll t) { // a随机数，u, t于外部计算
	ll ret = mod_pow(a, u, n);
	ll last = ret;
	for(int i=1; i<=t; ++i) {
		ret = mod_mult(ret, ret, n);
		if(ret == 1 && last != 1 && last != n-1) return true;
		last = ret;
	}
	if(ret!=1) return true; // a^(n-1)!=1(mod n)
	return false;
}

// Miller_Rabin()算法素数判定
//是素数返回true.(可能是伪素数，但概率极小，P=(1/2)^s)
//合数返回false;
// s:挑选s个随机数
bool Miller_Rabin(ll n, int s) {
	if(n==2) return true;
	if(n < 2 || !(n&1)) return false;
	ll u=n-1;
	ll t=0;
	while(!(u&1)) {u>>=1; ++t;} // 2^t*u = n-1
	for(int i=0; i<s; ++i) {
		ll a=rand()%(n-1)+1; // 若n为合数，证据为a的概率至少为1/2
		if(witness(a, n, u, t)) return false; // 合数
	}
	return true;
}

//************************************************
//pollard_rho 算法进行质因数分解
//************************************************
vector<ll> factors; // 质因数分解结果（无序）

inline ll gcd(ll a, ll b) {
	return b==0?a:gcd(b, a%b);
}

ll Pollard_rho(ll n, ll c) { // 伪随机函数f(x)=x^2+c，c为随机数
	ll i=1, k=2;
	ll x=rand()%n;
	ll y = x;
	while(1) {
		++i;
		x=(mod_mult(x, x, n)+c)%n; // 伪随机数
		ll d = gcd(y-x+n, n); // gcd注意负数
		if(d!=1 && d!=n) return d;
		if(y==x) return n; // 遇环退出
		if(i==k) {y=x; k<<=1;}
	}
}

// 对n进行递归素因子分解
void findfac(ll n) {
	if(Miller_Rabin(n, 20)) { // 素数
		factors.push_back(n); // 储存素因子
		return;
	}
	ll p = n;
	while(p>=n) p = Pollard_rho(p, rand()%(n-1)+1); // 找出当前合数的一个素因子
	findfac(p);
	findfac(n/p);
}

int T;
ll N;

void solve() {
	if(Miller_Rabin(N, 20)) {
		cout << "Prime" << endl;
	}
	else {
		factors.clear();
		findfac(N);
		cout << *min_element(factors.begin(), factors.end()) << endl;
	}
	// cout << "因子：\n";
	// for(vector<ll>::iterator i=factors.begin(); i!=factors.end(); ++i) {
		// printf("%lld ", *i);
	// }
	// cout <<endl;
}

int main()
{
#ifdef Oj
	freopen("1811.in", "r", stdin);
#endif
	cin >> T;
	while(T--) {
		cin >> N;
		solve();
	}
	return 0;
}
```

### POJ 2429
题目地址：[http://poj.org/problem?id=2429](http://poj.org/problem?id=2429)

#### 题目大意
给你两个数$a, b$，求他们的最大公约数$GCD$和最小公倍数$LCM$比较简单；那么给出$GCD$和$LCM$，求出$a, b$（升序），若有多解，求出$a+b$最小的那组解。

#### 思路
题目给的GCD, LCM都是64位的，暴力是行不通的。

$$\dfrac{a}{GCD}\dfrac{b}{GCD} = p\times q= \dfrac{LCM}{GCD}$$

那么用`Pollard_Rho`分解$\dfrac{LCM}{GCD}$求出所有质因子，然后在合并相同的质因子，最终搜索最小p+q，则$a=p\times GCD, b=q\times GCD$

#### AC代码
```cpp
/*************************************************************************
	> File Name: 2429.cpp
	  > Author: Netcan
	  > Blog: http://www.netcan.xyz
	  > Mail: 1469709759@qq.com
	  > Created Time: 2015-12-07 一 21:11:13 CST
 ************************************************************************/

#include <iostream>
#include <vector>
#include <string>
#include <queue>
#include <algorithm>
#include <cmath>
#include <ctime>
#include <cstdio>
#include <sstream>
#include <deque>
#include <functional>
#include <iterator>
#include <list>
#include <map>
#include <memory>
#include <stack>
#include <set>
#include <numeric>
#include <utility>
#include <cstring>
using namespace std;
typedef long long ll;
ll GCD, LCM;

ll mod_mult(ll a, ll b, ll mod) { // return	a*b%mod
	ll s = 0;
	while(b) {
		if(b&1) s=(s+a)%mod;
		a<<=1, b>>=1;
		if(a > mod) a -= mod;
	}
	return s;
}

ll mod_pow(ll x, ll n, ll mod) { // return a^b%mod
	ll res = 1;
	while(n) {
		if(n&1) res = mod_mult(res, x, mod);
		x=mod_mult(x, x, mod);
		n>>=1;
	}
	return res;
}
//以a为基,n-1=u*2^t      a^(n-1)=1(mod n)  验证n是不是合数
//一定是合数返回true,不一定返回false
bool witness(ll a, ll n, ll u, ll t) { // a随机数，u, t于外部计算
    ll ret = mod_pow(a, u, n);
    ll last = ret;
    for(int i=1; i<=t; ++i) {
        ret = mod_mult(ret, ret, n);
        if(ret == 1 && last != 1 && last != n-1) return true;
        last = ret;
    }
    if(ret!=1) return true; // a^(n-1)!=1(mod n)
    return false;
}

// Miller_Rabin()算法素数判定
//是素数返回true.(可能是伪素数，但概率极小，P=(1/2)^s)
//合数返回false;
// s:挑选s个随机数
bool Miller_Rabin(ll n, int s) {
    if(n==2) return true;
    if(n < 2 || !(n&1)) return false;
    ll u=n-1;
    ll t=0;
    while(!(u&1)) {u>>=1; ++t;} // 2^t*u = n-1
    for(int i=0; i<s; ++i) {
        ll a=rand()%(n-1)+1; // 若n为合数，证据为a的概率至少为1/2
        if(witness(a, n, u, t)) return false; // 合数
    }
    return true;
}

//************************************************
//pollard_rho 算法进行质因数分解
//************************************************
map<ll, int> factors; // 质因数分解结果（无序，每个质因子的n次方）

inline ll gcd(ll a, ll b) {
    return b==0?a:gcd(b, a%b);
}

ll Pollard_rho(ll n, ll c) { // 伪随机函数f(x)=x^2+c，c为随机数
    ll i=1, k=2;
    ll x=rand()%n;
    ll y = x;
    while(1) {
        ++i;
        x=(mod_mult(x, x, n)+c)%n; // 伪随机数
        ll d = gcd(y-x+n, n); // gcd注意负数
        if(d!=1 && d!=n) return d;
        if(y==x) return n; // 遇环退出
        if(i==k) {y=x; k<<=1;}
    }
}

// 对n进行递归素因子分解
void findfac(ll n) {
    if(Miller_Rabin(n, 20)) { // 素数
		++factors[n]; // 储存素因子，用map后面方便处理
        return;
    }
    ll p = n;
    while(p>=n) p = Pollard_rho(p, rand()%(n-1)+1); // 找出当前合数的一个素因子
    findfac(p);
    findfac(n/p);
}

void dfs(const vector<ll> &nf,const ll d, int i, ll& sum,  ll x, ll &a) { // 搜索最小的a+b，x为当前搜索的a，sum为储存最小和，a为最终答案
	if(i >= nf.size()) return;
	if(sum > d/x+x) { // 更新最小和
		a=x;
		sum = d/x+x;
	}
	dfs(nf, d, i+1, sum, x, a); // 不选nf[i]并搜索
	x*=nf[i]; // 选择nf[i]并搜索
	if(sum > d/x+x) { // 更新最小和
		a = x;
		sum = d/x+x;
	}
	dfs(nf, d, i+1, sum, x, a); // 这是选择nf[i]的情况
}

void solve() {
	factors.clear();
	vector<ll> nfactors; // 最终因子
	ll d = LCM/GCD; // (a/gcd)(b/gcd)=lcm/gcd，然后分解d，求出最终的(a/gcd), (b/gcd)
	if(d==1) {
		printf("%lld %lld\n", GCD, LCM);
		return;
	}
	findfac(d);

	for(map<ll, int>::iterator i = factors.begin(); i!=factors.end(); ++i)
		nfactors.push_back(mod_pow(i->first, i->second, (1ll<<63)-1)); // 对相同的质因子进行合并

	// for(vector<ll>::iterator i=nfactors.begin(); i!=nfactors.end(); ++i) {
		// printf("%lld ", *i);
	// }
	// cout << endl;
	ll a = 1;
	ll sum = a+d/a;
	dfs(nfactors, d, 0, sum, a, a);

	// for(int i=0; i<(1<<nfactors.size()); ++i) {
		// ll aa = 1;
		// for(int j=0; j<nfactors.size(); ++j) // 神奇的搜索。。。
			// if(i & (1<<j)) aa*=nfactors[j];

		// if(d/aa + aa < min_sum) {
			// min_sum = d/aa + aa;
			// a = aa;
		// }
	// }
	a = min(a, d/a);

	printf("%lld %lld\n", a*GCD, d/a*GCD);
}

int main()
{
#ifdef Oj
	freopen("2429.in", "r", stdin);
#endif
	while(cin >> GCD >> LCM) {
		solve();
	}
	return 0;
}
```

## 参考资料
1. 《算法导论》素数测试部分
2. [数论部分第一节：素数与素性测试](http://www.matrix67.com/blog/archives/234)
3. [笔记：A Quick Tutorial on Pollard's Rho Algorithm](http://www.ituring.com.cn/article/192557)
