---
layout: post
title: "线性回归"
categories: 机器学习
tags: [算法, 机器学习]
---

线性回归（Linear Regression）算法是机器学习中最简单的模型，计算简单，效果也还可以。学习它也有助于理解其他的机器学习算法。

## 问题描述

给定大小为m的n维数据集如下所示：

|     $y$     |     $x_1$     |     $x_2$     |  …   |     $x_n$     |
| :---------: | :-----------: | :-----------: | :--: | :-----------: |
|  $y^{(1)}$  |  $x^{(1)}_1$  | $x^{(1)}_{2}$ | ...  | $x^{(1)}_{n}$ |
|  $y^{(i)}$  | $x^{(2)}_{1}$ | $x^{(2)}_{2}$ | ...  | $x^{(2)}_{n}$ |
|     ...     |      ...      |      ...      | ...  |      ...      |
| ${y^{(m)}}$ | $x^{(m)}_{1}$ | $x^{(m)}_{2}$ | ...  | $x^{(m)}_{n}$ |

这里有几个概念：

* 特征（feature）：$ x_1, x_2, … , x_n $称作特征，比如我们要预测房屋价格，那么房屋面积、卧室数量都是特征。

* 特征向量（输入）：$x$，对于屋价格预测问题，一套房屋的信息就是一个特征向量，$x^{(i)}_j$表示第i个特征向量的第j个特征。

* 输出：$y^{(i)}$表示第i个特征向量对应的输出，也就是第i套房屋价格。

* 假设（hypothesis）：也称为决策函数、预测函数、回归方程。一个线性决策函数如下所示，表示我们认为数据集中的输入和输出可以用线性关系表示。$\theta$称作回归系数，决定了预测是否准确。

  $$ h_\theta(x) =\theta^Tx= \theta_0+\theta_1x_1+…+\theta_nx_n $$

* 代价函数（Cost Function）：也称为误差评估函数，用于评价学习效果，即真实值$y^{(i)}$和预测值$h_\theta(x^{(i)})$之间的差异。这里使用最小均方（Least Mean Square）来描述误差：

  $$ J(\theta_0, \theta_1,…,\theta_n) = \frac{1}{2m}\sum_{i=1}^m(h(x^{(i)})-y_i)^2 $$

  ​

## 梯度下降

在有了代价函数后，我们的目标就是求解回归系数$\theta$，使得代价函数足够小。最常见的手段是使用梯度下降（Gradient Descent）来逐步收敛。

梯度下降法描述为：


$$
\begin{align*}
& repeat\ until\ convergence\ \{\\
& \quad \theta_j:=\theta_j - \alpha\frac{\partial}{\partial\theta_j}J(\theta_1,\theta_2,...,\theta_n)\quad
for\ j=1\to m\\
& \}
\end{align*}
$$


其中，$\alpha$被称作学习率，表示$\theta$沿梯度方向行进的速率，$\alpha$太大容易造成梯度下降无法收敛，太小又会导致收敛速度太慢。在实际编程中，通常以一个基准值（比如0.1），然后3倍、10倍这样取值进行尝试。

$\frac{\partial}{\partial\theta_j}J(\theta_1,\theta_2,…,\theta_n)$表示对代价函数求$\theta_j$的偏导，那么对于上述最小均方代价函数的偏导为：



$$\frac{\partial}{\partial\theta_j}J(\theta_1,\theta_2,…,\theta_n)=\frac{1}{m}\sum^m_{i=1}(h(x^{(i)})-y_i) \times x^{(i)}_j$$



## 多项式回归

线性回归假定要求解的问题可以用线性关系拟合。但是实际情况下不是这样，这个时候可以采用多项式回归来构造更复杂的曲线。

将问题简化为一个1维向量，假设我们的回归方程是一个三阶函数：




$$
h_\theta(x) = \theta_0 + \theta_1 x_1 + \theta_1 x_1^2 + \theta_3 x_1^3
$$


这个时候可以创建两个新特征$x_2=x_1^2$、$x_3=x_1^3$，那么就一个多项式回归问题转换成线性回归问题。甚至可以用平方根函数来表示我们的回归方程：




$$
h_\theta(x) = \theta_0 + \theta_1 x_1 + \theta_2\sqrt{x_1}
$$


这里本质上是通过特征变换的方式将低维空间转换到高维空间。但是要注意的是，在这种情况下更要注意特征缩放，因为函数的阶导致特征放大，梯度下降的迭代过程更难收敛。

## 正规方程

相较于梯度下降逐步迭代逼近最优解，正规方程可以一次性求解参数$\theta$，在某些情况下这是更好的求解方法。

要用正规方程求解，首先定义矩阵如下：


$$
Y=
\left\{
 \begin{matrix}
y^{(1)}\\
y^{(2)}\\
\vdots\\
y^{(m)}
  \end{matrix}
  \right\} 
  \quad
  X=
\left\{
 \begin{matrix}
1&x^{(1)}_{1}&x^{(1)}_{2}&\cdots&x^{(1)}_{n}\\
1&x^{(1)}_{1}&x^{(2)}_{2}&\cdots&x^{(2)}_{n}\\
\vdots&\vdots&\vdots&\ddots&\vdots\\
1&x^{(m)}_{1}&x^{(m)}_{2}&\cdots&x^{(m)}_{n}
\end{matrix}
\right\}
\\[2ex]
$$


则需要求解的目标矩阵$\theta$可以通过以下公式计算：


$$
\theta=(X^TX)^{-1}X^TY
$$


正规方程法不需要选择$\alpha$，不需要迭代，但是有着较高的算法复杂度（矩阵转置的复杂度是$O(n^3)$）。当n偏大时计算可能就比较慢，一般来说n超过10000就可以考虑采用梯度下降法。

## 特征缩放

在梯度下降求解回归方程中，如果数据集中特征的范围过大或过小，就会影响收敛速度，所以要将特征标准化处理到[0,1]或者[-1,1]之间。



### Min-max scaling

Min-max scalling是最简单的方式，也称作normalisation，缩放后特征将在[0,1]之间，公式如下：


$$
x_i:=\frac{x_i-min(x_i)}{max(x_i)-min(x_i)}
$$


其中$max$和$min$分别是特征$x_i$的均值、最大值和最小值。

### Mean normalisation

Mean normalisation和min-max scaling类似，区别只在于以特征$x_i$的平均值$mean(x_i)$为中心，缩放后特征范围将在[-1,1]之间，公式如下：


$$
x_i:=\frac{x_i-mean(x_i)}{max(x_i) - min(x_i)}
$$


### Standardization

特征standardization缩放后，将满足正态分布，公式如下：


$$
x_i:=\frac{x_i-\mu}{\delta}
$$

其中$\mu$和$\delta$分别是特征$x_i$的均值和标准差，缩放后的特征在[-1,1]之间。

## 参考资料

* [斯坦福大学公开课 ：机器学习课程](http://open.163.com/special/opencourse/machinelearning.html)