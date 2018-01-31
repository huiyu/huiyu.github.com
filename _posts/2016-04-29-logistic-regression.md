---
layout: post
title: "逻辑回归"
categories: 机器学习
tags: [算法, 机器学习]
---

逻辑回归是（Logistic Regression，LR）是机器学习中的一个分类模型，由于其简单高效，在实际生产环境中有着非常广泛的应用。

## 模型描述

### Hypothesis

逻辑回归的决策函数（Hypothesis）实际上就是在线性回归的基础上套用了一个逻辑函数：


$$
h_\theta=g(\theta^Tx)=\frac{1}{1+e^{-\theta^Tx}}
$$


其中，函数$g(z)=\frac{1}{1+e^{-z}}$称作逻辑函数（Logistic Function）或者S函数（Sigmoid Function），其图形如下所示：



![Sigmoid Function]({{ site.baseurl }}/images/logistic-regression/sigmoid-function.png)



函数$h_\theta$的意义在于给定了输出是否为1的概率，记作$h_\theta(x)=P(y=1 &#124; x;\theta)=1-P(y=0 &#124; x;\theta) $。比如$h_\theta(x)=0.7$表示结果是1的概率是0.7。对于一般的分类问题，选择阈值0.5是通常的做法，即大于等于0.5即判断为1。实际情况可以选择不同的阈值，比如对于正例判断要求更高，就可以选择高一点的阈值。



### Cost Function

逻辑回归的决策函数和线性回归不同，套用线性回归的代价函数会导致代价函数不是凸函数。因此逻辑回归的代价函数定义为：


$$
J(\theta)=\frac{1}{m}\sum^m_{i=1}Cost(h_\theta(x^{(i)}-y^{(i)})) \\
Cost(h_\theta(x),y)=
\begin{cases}
 -log(h_\theta(x))   &{if\ y=1}\\
-log(1-h_\theta(x)) &{if\ y=0}\\
\end{cases}
$$

函数可以简化为：


$$
\begin{equation}
J(\theta)=-\frac{1}{m}\sum^m_{i=1} \left[ y^{(i)}log(h_\theta(x^{(i)})) + (1-y^{(i)})log(1-h_\theta(x^{(i)})) \right]
\end{equation}
$$

上述公式的向量化表达：


$$
h=g(X\theta) \\
J(\theta)=\frac{1}{m}(-y^Tlog(h)-(1-y)^Tlog(1-h))
$$




## 参数求解

模型确定后，剩下就是求解回归系数$\theta$。同样地逻辑回归可以采用梯度下降方式方式求解：


$$
\begin{align*}
& repeat\ until\ convergence\ \{\\
& \quad \theta_j:=\theta_j-\alpha\frac{\partial}{\partial\theta_j}J(\theta)\\
& \}
\end{align*}
$$



其中，偏导数$ \frac{\partial}{\partial{\theta_j}} J(\theta)=\frac{1}{m}\sum_{i=1}^m(h_\theta(x^{(i)})-y^{(i)}) x_j^{(i)}$，这和线性回归的形式是一样的。

上述公式的向量化表达为：


$$
\theta:=\theta-\frac{\alpha}{m}X^T(g(X\theta) - \vec{y})
$$


此外，除了梯度下降，其他常用的凸优化的方法都可以解决该问题，如共轭梯度下降，牛顿法，LBFGS等。



## 多分类

如果目标不是01分类问题，而是有多个类别，那么此时问题就变成一个多分类问题。最简单的方式是对每一个类别训练一个二元分类器（one-vs-all），通过多个二元分类器组合判断是否属于哪个类别；此外当多个类别之间是互斥关系时，可以采用[softmax回归](http://ufldl.stanford.edu/wiki/index.php/Softmax%E5%9B%9E%E5%BD%92)，这里就不再详述。



## 参考资料

* [斯坦福大学公开课 ：机器学习课程](http://open.163.com/special/opencourse/machinelearning.html)
* [美图点评技术团队：Logistic Regression 模型简介](https://tech.meituan.com/intro_to_logistic_regression.html)