---
title: TensorFlow实战（一）：简单示例
date: 2019-06-11 15:28:30
tags: TensorFlow
categories: Technology
---

本文是TensorFlow实战的开篇，请确认好您已经安装好Python3开发环境，并安装设置好NumPy、matplotlib、TensorFlow和Jupyter Notebook。

# 软件介绍

## Jupyter Notebook

## matplotlib

matplotlib是一个绘图库，它允许用户使用Python创绩动态的、自定义的可视化结果。它与Numpy无缝集成，其绘图结果可直接显示在Jupyter Notebook中。matplotlib也可将数值以图像的形式可视化，这个功能可用于验证图像识别任务的输出，并将神经网络的内部单元可视化。构建在matplotlib之上的其它层，如Seaborn，可用于增强其功能。

## Numpy

# 代码示例

以下代码可以运行在Jupyter Notebook中：

```python
import tensorflow as tf
import numpy as np
import matplotlib.pyplot as plt
%matplotlib inline
a= tf.random_normal([2,20])
sess=tf.Session()
out=sess.run(a)
x,y=out
plt.scatter(x,y)
plt.show()
```

成功运行如下图：

{% asset_img  python.png 运行示例  %}

我们逐条讲解代码：

1. 用TensorFlow定义一个由随机数组成的2*20矩阵，比将其赋给变量a。
2. 启动tensorFlow Session，并将其赋给一个sess对象。
3. 用sess.run()方法执行对象a，并将输出（NumPy数组）赋给out。
4. 将这个2*20的矩阵划分成为两个1 *10的向量x和y。
5. 利用pyplot模块绘制散点图，x对应横轴，y对应纵轴。