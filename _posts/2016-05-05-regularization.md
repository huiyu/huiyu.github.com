---
layout: post
title: "利用正规化解决过拟合的问题"
categories: "机器学习"
tags: [算法, 机器学习, 正则化, 过拟合]
---

评价机器学习模型的一个重要指标就是模型的泛化能力，表现在训练集和测试集上都有良好的适应性。如果在训练集和测试集上表现都很差，可能是欠拟合（under fitting）导致。如果模型在训练集上表现的很好，却在训练集上表现很差，那通常就是过拟合（over fitting）。解决过拟合问题有很多种方法，其中正规化（regularization）是改善过拟合问题的常见方法。

正规化是这样一个思路，在代价函数（cost function）中加入“惩罚项”，使得训练得出的回归系数$\theta$足够小，从而拉升曲线使之更为平滑。

在线性回归中，我们修改代价函数为：


$$
\begin{equation}
J(\theta_0, \theta_1,…,\theta_n) = \frac{1}{2m} \left[ \sum_{i=1}^m(h(x^{(i)})-y_i)^2 +\lambda\sum^n_{i=1}\theta^2_j\right]
\end{equation}
$$


其中项$\lambda\sum^n_{i=1}\theta^2_j$称作正规化项，也就是所谓的惩罚项，这里通常我们省略常数项$\theta_0$。参数$\lambda$是个被称作正则化参数，越大惩罚力度也就越大，但是如过大就有可能出现欠拟合的情况。

在这个代价函数的基础上，就可以采用梯度下降等凸优化的方法来求解回归系数，这和原先的步骤没有任何区别。对于逻辑回归做法也是类似的，加入正规化参数的代价函数如下所示：


$$
J(\theta)=-\frac{1}{m}\sum^m_{i=1}\left [y^{(i)}log(h_\theta(x^{(i)})) + (1-y^{(i)})log(1-h_\theta(x^{(i)})) \right ] +\frac{\lambda}{ m}\sum^n_{i=1}\theta^2_j
$$


线性回归除了梯度下降，还有正规方程来求解回归系数。此时需要对公式做一点修改：


$$
\begin{equation}
\theta=(X^TX+\lambda L)^{-1}X^TY \quad 
where\ L=\left\{\begin{matrix} 
0 &  &  &  &  \\
   &1&  &  &  \\
   &  &1&  &  \\
   &  &  & \ddots & \\
   &  &  &  & 1 \\
\end{matrix} \right\} \\
\end{equation}
$$

## 参考资料

* [斯坦福大学公开课 ：机器学习课程](http://open.163.com/special/opencourse/machinelearning.html)
* [机器学习中如何解决过拟合](https://zhuanlan.zhihu.com/p/29494325)