title: c语言解决汉诺塔问题
toc: false
original: true
tags:
  - C/C++
  - 算法
categories:
  - 编程
date: 2015-01-23 17:05:28
updated: 2015-01-23 17:05:28
---
游戏规则:
有三根相邻的柱子，标号为A,B,C，A柱子上从下到上按金字塔状叠放着n个不同大小的圆盘，编号从上往下1到n，现在把所有盘子一个一个移动到柱子C上，并且每次移动同一根柱子上都不能出现大盘子在小盘子上方。
代码：
```cpp
/*************************************************************************
    > File Name: hanoi.c
    > Author: Netcan
    > Mail: 1469709759@qq.com
    > Created Time: 2015/1/23 16:32:17
 ************************************************************************/

/* 盘子从上往下编号，1到n
 * 当 n == 1 的时候，
 *      直接将盘中从“A”柱移到“C”柱
 * 当 n >= 2 的时候，
 *      先将前n-1个盘子从“A”柱借助“C”柱移到“B”柱
 *      然后，将第n个盘子从“A”柱移到“C”柱
 *      最后，将“B”柱上n-1个盘子从“B”柱借助“A”柱移到“C”柱
 */

void hanoi(int n, char A, char B, char C); // 表示n个盘子从“A”柱借助“B”柱移到“C”柱


#include<stdio.h>
int main()
{
    int n;
    printf("输入盘子数量：");
    scanf("%d",&n);
    hanoi(n, 'A', 'B', 'C');
    return 0;
}

void hanoi(int n, char A, char B, char C)
{
    static int steps = 0;
    if ( 1 == n )
        printf("第%d步: 将编号为%d的盘子从%c柱移到%c柱\n", ++steps, n, A, C);
    else {
        hanoi(n-1, A, C, B);
        printf("第%d步: 将编号为%d的盘子从%c柱移到%c柱\n", ++steps, n, A, C);
        hanoi(n-1, B, A, C);
    }
    return;
}
```
