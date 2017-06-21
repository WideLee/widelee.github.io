---
layout:     post
title:      "逻辑回归及其并行化计算"
subtitle:   ""
date:       2017-06-21
author:     "SmartYi"
header-img: "img/in-post/logistic-regression/head.jpg"
tags:
    - 机器学习
---

## 逻辑回归及其并行化计算

> Created  By MingkuanLi At 2017-06-21

### 1. 逻辑回归简介

逻辑回归（Logistic Regression）常用于机器学习中的二分类问题里，例如垃圾邮件的识别等。在实际情况中，可能由于数据规模庞大，包括大数据量以及高数据维度，导致单线程进行回归受到效率的限制。在本文中，首先介绍逻辑回归的Cost函数以及使用梯度下降的方式最小化Cost函数，然后根据梯度的求解方法，给出一个并行化的策略。

### 2. 逻辑回归Cost函数

假设有 $M$ 个训练样本，分别是$(X_1, y_1), (X_2, y_2), \dots, (X_M, y_M)$，其中$y_i = \{-1, 1\}$，$y = -1$ 代表负样本，$y = 1$ 代表正样本。每个$X_i = \{x_1, x_2, \dots, x_N\}$为一个 $N$ 维的特征向量。

在逻辑回归模型中，使用Sigmoid函数把值压缩到$0$~$1$之间。Sigmoid函数的表达式为$\sigma(x) = \frac{1}{1 + e^{-x}}$，函数图像如下图所示：

![Sigmoid图像](/img/in-post/logistic-regression/1.png)

假设 $W$ 是 $N$ 维的特征权重向量，那么第 $j$ 个样本为正样本的概率是：

$$
P\left(y_j = 1|W, X_j\right) = \frac{1}{1 + e^{-W^T X_j}}
$$

``` python
# 注意：测试各个矩阵的大小分别如下：
X.shape = (M, N)
y.shape = (M, 1)
W.shape = (N, 1)
```

求解逻辑回归的问题实际上是要找到一个合适的 $W$, 使得在正样本里 $P\left(y_j = 1\|W, X_j\right)$ 尽可能大，在负样本里尽可能小，因此巧妙地把 $y_i$ 作为指数的一个符号位，联合起来得到如下目标：

$$
\max\limits_W{p(W)} = \prod\limits_{j=1}^M{\frac{1}{1 + e^{-y_iW^T X_j}}}
$$

为了方便计算，对上式求 $log$ 并取负号，得到目标函数：

$$
\min\limits_{W}{f(W)} = \sum\limits_{j=1}^{M}{\log\left(1 + e^{-y_iW^T X_j}\right)}
$$

### 3. 使用梯度下降迭代
使用梯度下降的算法求解这个最小化Cost函数的问题，最重要的是求解梯度，假设在第 $t$次迭代的梯度：

$$
\begin{array}{rl}
G_t = \triangledown_W f(W) &=  \sum\limits_{j=1}^{M}{\frac{1}{1 + e^{-y_iW^T X_j} } \cdot e^{-y_iW^T X_j} \cdot \left(-y_jX_j\right)} \\
&= \sum\limits_{j=1}^{M}{\left(\frac{1}{1 + e^{-y_iW^T X_j}}- 1\right)y_jX_j}
\end{array}
$$

使用矩阵的形式可以表示为：

$$
G_t = X^T \left[\left(\frac{1}{1 + e^{-y * (X W)}}- 1\right) * y\right]
$$

其中，$A * B$ 表示矩阵对应项相乘，$A$和$B$的维度应该相同，而且计算结果的维度也应该一样，代码如下：
```python
import numpy as np

def sigmoid(self, x):
  return 1 / (1 + np.exp(-x))

def calculate_grad(X, W, Y):
  d = np.matmul(X, W)

  sigmoid_val = sigmoid(np.multiply(Y, d)) - 1
  multiply_val = np.multiply(sigmoid_val, Y)
  g = np.matmul(X.T, multiply_val)

```

### 4. 并行化梯度下降

#### 4.1 数据分割
（To Do…）

#### 4.2 并行运算
（To Do…）

### 5. 遇到的问题

问题是对于数据特征维度$N$如果开多个线程/进程进行运算，耗时和使用单线程相近，而对于数据量$M$开多个线程/进程运算的时候，时间会大大增加

- 一开始以为是使用Python的threading进行并行运算的问题，可能因为Python的全局解释器锁（Global Interpreter Lock）导致，在使用PyCharm分析线程运行时间可以发现，时间较长的并行方案中有一段是Waiting Lock的状态
- 后来改用了multiprocessing的并行库进行测试，在分析代码的时候发现`numpy.matmul`函数的运行时间很奇怪，单进程的时候执行100000x1000规模的矩阵乘法耗时0.2s左右，而并行使用两个进程同时执行50000x1000规模的矩阵乘法的时候，两个进程该函数执行的时间都为0.8s左右，**暂时还没找到`matmul`函数多进程的时候速度反而变慢的原因**。

### 6. 参考博客
- (道可叨 | python 线程，GIL 和 ctypes)[http://zhuoqiang.me/python-thread-gil-and-ctypes.html]
- (详解并行逻辑回归)[http://www.csdn.net/article/2014-02-13/2818400-2014-02-13]
- (python 多线程编程)[http://blog.csdn.net/guopengzhang/article/details/5458091]
- (Python线程同步机制: Locks, RLocks, Semaphores, Conditions, Events和Queues - Zhou's Blog)[http://yoyzhou.github.io/blog/2013/02/28/python-threads-synchronization-locks/]
- (理解Python并发编程一篇就够了 - 进程篇)[http://www.dongwm.com/archives/%E4%BD%BF%E7%94%A8Python%E8%BF%9B%E8%A1%8C%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B%E7%AF%87/]
- (Andrew Ng logistic-regression keynote)[/img/in-post/logistic-regression/Lecture6.pptx]
- (Andrew Ng logistic-regression Lecture Note)[/img/in-post/logistic-regression/loss-functions.pdf]
