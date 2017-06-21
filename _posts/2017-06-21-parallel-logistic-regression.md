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

假设有 $M$ 个训练样本，分别是 $(X_1, y_1), (X_2, y_2), \dots, (X_M, y_M)$ ，其中 $y_i = \{-1, 1\}$，$-1$ 代表负样本，$1$ 代表正样本。每个 $X_i = \{x_1, x_2, \dots, x_N\}$ 为一个 $N$ 维的特征向量。


### 3. 使用梯度下降迭代

### 4. 并行化梯度下降

### 5. 遇到的问题
