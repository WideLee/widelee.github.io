---
layout:     post
title:      "TensorFlow Programmer's Guide翻译"
subtitle:   ""
date:       2018-01-18
author:     "SmartYi"
header-img: "img/in-post/tensorflow-guide/head.jpeg"
tags:
    - TensorFlow
    - AI
---

## TensroFlow Programmer's Guide

* 目录
{:toc}
### 1. 概述

学习TensorFlow顺便翻译一下官方的TensorFlow Programmer's Guide，也帮助自己理解使用TensorFlow完成深度学习任务的具体流程以及每一部分需要到的技术，有什么翻译或者理解错的地方欢迎评论指出。

> 英文原文地址：[https://www.tensorflow.org/programmers_guide/](https://www.tensorflow.org/programmers_guide/)

文档在r1.3版本中做了大量的修改，下面分为以下几个部分介绍TensorFlow：

- `Estimators`， 引入了一个高级的TensorFlow API，大大简化了机器学习的编程复杂性
- `Tensors`，它是TensorFlow中最基本的对象，他解释了如何创建，操作，以及访问Tensor
- `Variables`，详细介绍如何在程序中表示共享的持久状态（Shared, Persistent state）
- `Graphs and Sessions`介绍：
  - 数据流图，TensorFlow将计算表示为运算符（Operation）之间的一种依赖关系
  - Session是TensorFlow在一个或多个本地或远程设备上运行数据流图的机制。如果使用TensorFlow的低级API，那么这是一个很基础的部分。但如果使用高级的TensorFlow API，例如Estimators或者Keras，这些高级的API会帮你创建和维护Session，但了解数据流图以及Session还是很有帮助的
- `Saving and Restoring`介绍如何保存于恢复变量以及模型
- `Input Pipelines`介绍如何设置数据管道将数据集读入TensorFlow程序中
- `Embeddings`介绍了嵌入的概念，提供了一个在TensorFlow中嵌入训练的简单例子，以及描述了如何在TensorBoard中进行可视化
- 调试TensorFlow程序介绍如何使用`tfdbg`对TensorFlow程序进行调试
- TensorFlow版本兼容性，这解释了向后兼容性

### 2. Estimators

`Estimators`是Tensorflow的一个高级的API，极大地简化了机器学习编程，它主要封装了以下几个操作：

- 训练`Training`
- 评估`Evaluation`
- 预测`Prediction`
- 导出模型`Export for serving`

你可以使用我们提供的`Estimators`，也可以自己编写`Estimators`，所有的`Estimators`都是基于[`tf.estimator.Estimator`](https://www.tensorflow.org/api_docs/python/tf/estimator/Estimator)类的

#### 2.1 使用Estimators的优点

`Estimators`提供了如下的优点：

- 你可以在本地主机上或者分布式多服务器环境中运行`Estimators`模型，而无需更改模型。此外，在CPU，GPU或TPU上运行都不需要对模型进行重新编码
- `Estimators`对于模型开发人员来说简化了模型的共享
- 可以使用高级直观的代码开发更前沿的模型。简而言之，使用`Estimators`创建模型通常比使用低级别的TensorFlow API更容易。
- `Estimators`本身是建立在tf.layers上，这简化了自定义的过程
- `Estimators`自动建立计算流图，换句话说，你不需要重新建立计算图
- `Estimators`提供了安全的分布式训练控制，例如如何以及何时执行下列操作：
  - 构建计算流图
  - 初始化变量
  - 开始执行队列（start queues）
  - 处理异常
  - 创建记录点Checkpoints以及从出现错误的记录点中恢复
  - 保存训练过程summary用于TensorBoard的可视化

在使用`Estimators`编写应用程序时，必须将数据输入管道与模型分开。这种分离简化了不同数据集上进行实验。

#### 2.2 使用预定义的Estimators

使用预定义的Estimators可以在比低级TensorFlow更高层的概念层面上进行编码，不需要担心创建计算流图以及Session，因为`Estimators`会帮助你处理好所有的流水线。也就是说，预定义的`Estimators`会为你创建Graph以及Session对象，此外，他让你只需要进行最少量代码的更改既可以尝试不同的模型架构。例如，`DNNClassifier`是一个预定义的`Estimators`类，他通过dense feed-forward神经网络对分类模型进行训练。

依赖预定义的`Estimators`的TensorFlow程序通常有以下四个步骤：

1. **写一个或多个导入数据库（dataset）的函数**：例如，需要创建一个函数导入训练集数据，另外一个函数导入测试集数据，每个数据导入函数必须返回两个object：

   - 一个字典，其中key是特征的名称（label），值是一个Tensor表示对应的数据
   - 包含一个或多个标签的Tensor

   例如，以下代码演示了数据导入函数的基本框架：

   ```python
   def input_fn(dataset):
      ...  # manipulate dataset, extracting feature names and the label
      return feature_dict, label
   ```

2. **定义输入特征每一列的含义**：每个tf.feature_column表示一个特征的名称，类型还有其他的预处理输入。例如，下面的代码创建了三个保存整数或者浮点数的特征列，前两个特征列只是表示特征的名称与类型，第三个特征列指定了一个lambda程序用来缩放原始数据

   ```python
   # Define three numeric feature columns.
   population = tf.feature_column.numeric_column('population')
   crime_rate = tf.feature_column.numeric_column('crime_rate')
   median_education = tf.feature_column.numeric_column('median_education',
                       normalizer_fn='lambda x: x - global_education_mean')
   ```

3. **实例化相应的Estimators**：例如，下面的例子实例化了一个叫`LinearClassifier`的预定义`Estimators`

   ```python
   # Instantiate an estimator, passing the feature columns.
   estimator = tf.estimator.Estimator.LinearClassifier(
       feature_columns=[population, crime_rate, median_education],
       )
   ```

4. **调用训练，评估以及inference方法**：例如，所有的Estimators都提供了一个train方法，用来训练模型。

   ```python
   # my_training_set is the function created in Step 1
   estimator.train(input_fn=my_training_set, steps=2000)
   ```

使用预定义的`Estimators`在实践中提供了如下的优点：

- 确定计算图的不同部分应该在哪里运行，在单个机器上以及集群上实施不同的策略
- 记录一些有用的摘要/中间变量用于可视化显示

如果你不使用预定义的`Estimators`，那么上述的特性必须自己实现

无论是预定义的还是自己实现的Estimators，核心都是他的model函数，这个函数用来为训练，评估与预测构建数据流图。当您使用预制的Estimators时，其他人已经实施了模型功能。当依靠自定义的Estimators时，您必须自己编写模型函数。[这里](https://www.tensorflow.org/extend/estimators)详细描述了模型函数是如何编写的。

使用Estimators推荐的工作流如下：

1. 假设合适的预定义Estimators存在，那么用他来构建模型并使用他的结果建立后续的基本。
2. 使用预定义Estimators构建以及测试适合自己的整体流水线，包括数据的完整性与可靠性
3. 如果有多个预定义的Estimators适用，那么进行实验来选择最好的结果
4. 如果可能的话，使用自定义的Estimators提高自己的模型

#### 2.3 使用Keras模型建立Estimators

你可以把已经存在的Keras模型转换成Estimators，这样可以使得Keras模型拥有Estimators的功能，例如分布式训练等。如下例子通过调用[`tf.keras.estimator.model_to_estimator`](https://www.tensorflow.org/api_docs/python/tf/keras/estimator/model_to_estimator) 实现转换：

```python
# Instantiate a Keras inception v3 model.
keras_inception_v3 = tf.keras.applications.inception_v3.InceptionV3(weights=None)
# Compile model with the optimizer, loss, and metrics you'd like to train with.
keras_inception_v3.compile(optimizer=tf.keras.optimizers.SGD(lr=0.0001, momentum=0.9),
                          loss='categorical_crossentropy',
                          metric='accuracy')
# Create an Estimator from the compiled Keras model.
est_inception_v3 = tf.keras.estimator.model_to_estimator(keras_model=keras_inception_v3)
# Treat the derived Estimator as you would any other Estimator. For example,
# the following derived Estimator calls the train method:
est_inception_v3.train(input_fn=my_training_set, steps=2000)
```

### 3. Tensor

正如名称所示，TensorFlow是定义和运行设计Tensor的计算框架，Tensor是向量与矩阵想可能更高维度的推广。实际上，TensorFlow将Tensor表示为基本数据类型的n维数组。

当编写一个TensorFlow程序时，我们操作和传递的主要对象是`tf.Tensor`，一个`tf.Tensor`对象表示一个定义好的计算，在运行阶段会产生一个值。TensorFlow程序首先建立`tf.Tensor`对象的计算图，详细说明如何根据其他Tensor计算每个Tensor的值，然后通过运行这个图形的一部分来实现计算得到最终的结果。

一个`tf.Tensor`有如下两个特性

- 数据类型（例如float32, int32, string）
- 形状（Shape）

有一些Tensor是特殊的，这些会在其他部分介绍，主要有：

- `tf.Variable`
- `tf.Constant`
- `tf.Placeholder`
- `tf.SparseTensor`

除了`tf.Variable`之外，Tensor的值都是不变的，这意味着单个Tensor在执行后中它的值是固定的。但是，两次评估相同的Tensor可以返回不同的值，例如Tensor可以是从磁盘中读取数据或生成随机数的结果。

#### 3.1 秩（Rank）

一个`tf.Tensor`的秩为它的维数，请注意，TensorFlow里的rank与矩阵的rank的含义并不相同。如下表所示，TensorFlow中的各个不同的rank对应于不同的数学含义

| Rank | 数学含义                      |
| ---- | ------------------------- |
| 0    | 标量（只有大小）                  |
| 1    | 向量（有大小和方向）                |
| 2    | 矩阵                        |
| 3    | 3-Tensor（cube of numbers） |
| n    | n-Tensor                  |

##### Rank 0

下面的片段创建了一些rank为0的Tensor：

```python
mammal = tf.Variable("Elephant", tf.string)
ignition = tf.Variable(451, tf.int16)
floating = tf.Variable(3.14159265359, tf.float64)
its_complicated = tf.Variable((12.3, -4.85), tf.complex64)
```

##### Rank 1

创建Rank 1的Tensor，我们需要传一个list作为初始值，例如：

```python
mystr = tf.Variable(["Hello"], tf.string)
cool_numbers  = tf.Variable([3.14159, 2.71828], tf.float32)
first_primes = tf.Variable([2, 3, 5, 7, 11], tf.int32)
its_very_complicated = tf.Variable([(12.3, -4.85), (7.5, -6.23)], tf.complex64)
```

##### 更高维的Tensor

一个rank为2的Tensor由至少一行和至少一列组成：

```python
mymat = tf.Variable([[7],[11]], tf.int16)
myxor = tf.Variable([[False, True],[True, False]], tf.bool)
linear_squares = tf.Variable([[4], [9], [16], [25]], tf.int32)
squarish_squares = tf.Variable([ [4, 9], [16, 25] ], tf.int32)
rank_of_squares = tf.rank(squarish_squares)
mymatC = tf.Variable([[7],[11]], tf.int32)
```

更高维的Tensor同样由一个n维数组构成，例如，在图像处理过程中，使用许多rank为4的Tensor，对应一个batch的样本，图像的宽度，高度以及色彩通道。

```python
my_image = tf.zeros([10, 299, 299, 3])  # batch x height x width x color
```

##### 获取一个 `tf.Tensor` 对象的rank

通过`tf.rank`方法可以获取一个`tf.Tensor`对象的rank，例如，下面的代码片段获取上一节中定义的`tf.Tensor`的rank：

```python
r = tf.rank(my3d)
# After the graph runs, r will hold the value 3.
```



> 官方中文翻译见：https://www.tensorflow.org/programmers_guide/tensors



