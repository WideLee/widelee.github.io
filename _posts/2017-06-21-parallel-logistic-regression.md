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

假设有 $M$ 个训练样本，分别是 $(X_1, y_1), (X_2, y_2), \dots, (X_M, y_M)$ ，其中 $y_i = \{-1, 1\}$，$y = -1$ 代表负样本，$y = 1$ 代表正样本。每个 $X_i = \{x_1, x_2, \dots, x_N\}$ 为一个 $N$ 维的特征向量。

在逻辑回归模型中，使用Sigmoid函数把值压缩到 $0$~$1$ 之间。Sigmoid函数的表达式为 $\sigma(x) = \frac{1}{1 + e^{-x}}$，函数图像如下图所示：

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

求解逻辑回归的问题实际上是要找到一个合适的 $W$, 使得在正样本里 $P\left(y_j = 1|W, X_j\right)$ 尽可能大，在负样本里尽可能小，因此巧妙地把 $y_i$ 作为指数的一个符号位，联合起来得到如下目标：

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

### 5. 遇到的问题
