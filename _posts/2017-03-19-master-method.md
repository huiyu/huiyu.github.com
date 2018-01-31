---
layout: post
title: "使用主方法求解递归式的时间复杂度"
categories: 算法
tags: [算法, 递归, 复杂度]
---

在计算算法的时间复杂度时，通常需要求解递归式。求解解递归式主要有三种方法：1）代入法：猜测解的形式，然后用数学归纳法证明解的正确性；2）递归树法：画出递归树，并对树中每层的代价求和得到总代价；3）主方法（master method）：也就是本文要讨论的方法。

假设递归式可以表示成以下形式：

$$
T(n)=aT(n/b)+\Theta(n^d)
$$

其中$$a\ge1$$和$$b>1$$是常数，$$\Theta(n^d)$$是n的d阶渐进正函数。那么$$T(n)$$有如下的渐进紧确界：

$$
\begin{equation}
T(n)=\begin{cases}
\Theta(n^dlogn) & if\ a=b^d\\
\Theta(n^d)  & if\ a<b^d\\
\Theta(n^{log_{b}a}) & if\ a>b^d
\end{cases}
\end{equation}
$$

这就是主方法公式，使用它非常简单，来看两个例子。

1）归并排序的分治算法递归式为：

$$
T(n)=2T(n/2)+\Theta(n)
$$

此时$a=2，b=2，d=1$，计算得出$a=2=b^d=1$，那么归并排序的复杂度$T(n)=\Theta(n^dlogn)=\Theta(nlogn)$。

2）二分查找的递归式为：

$$
T(n)=T(n/2)+\Theta(1)
$$

此时$a=1，b=2，d=0$，计算得出$a=1=b^d=1$，那么二分查找的复杂度$T(n)=\Theta(n^dlogn)=\Theta(logn)$。

## 参考资料

* [算法导论](https://book.douban.com/subject/1885170/)

