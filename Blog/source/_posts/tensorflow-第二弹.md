---
title: tensorflow-第二弹
date: 2017-07-20 20:37:26
tags: tensorflow学习
categories: 学习
mathjax: true
---
刚刚研究了一下以前写的tensorflow发现就开了个头，而且都是两个月之前的事情了，现在看起来作为入门介绍起来也是一件比较简单的事情了，这一段时间按照官网的教程也写了一个mnist识别的cnn的例子，当然咯自己也写了一个自己的图像训练的代码，只是目前还没有开始训练......但是按照官网教程写的东西毕竟只是依葫芦划瓢而已并没有深入理解，这一次慢慢去深入理解tensorflow的机制。其实再tenflow的情况下主要分为两个操作，一个是构建Graph，Graph由很多op，和tensorflow构成，好了今天我们从一个简单的线性回归的例子开始进行深入的剖析。  
线性回归可以简单的表示为$y=a\cdot x+b$的形式，那么我们在获取了一系列的x，y数据的情况下可以拟合得到线性方程，而线性拟合通常直接求解，但是我们为了说明tensorflow的原理所以强行用梯度下降算法进行迭代拟合求解，废话不多说，下面我们构建起线性拟合的Graph：
```python
import tensorflow as tf
import numpy as np
from numpy.random import RandomState

with tf.name_scope('input'):
    x = tf.placeholder(tf.float32,name = 'xinput')
    y = tf.placeholder(tf.float32,name = 'yinput')
    tf.summary.scalar('x',x)
    tf.summary.scalar('y',y)

with tf.name_scope('parameter'):
    a = tf.Variable(0.0,name='a')
    b = tf.Variable(0.0,name='b')
    tf.summary.scalar('a',a)
    tf.summary.scalar('b',b)

with tf.name_scope('predict'):
    y_=a*x+b;

with tf.name_scope('loss'):
    loss = tf.square((y_-y),name='loss')
    tf.summary.scalar('loss',loss)

with tf.name_scope('train_step'):
    train_step = tf.train.GradientDescentOptimizer(0.01).minimize(loss)
    #tf.summary.scalar('train_step',train_step)

merged = tf.summary.merge_all()
#data prepare
xdata = np.linspace(0,0.5*np.pi,3000)
ydata = 0.4*xdata+0.5;
rdm = RandomState(1)

with tf.Session() as sess:
    writer = tf.summary.FileWriter('tensorflow-start_Log', sess.graph)
    sess.run(tf.global_variables_initializer())
    for i in range(2000):
        idx = rdm.randint(0,3000)
        summary,_=sess.run([merged,train_step],feed_dict={x:xdata[idx],y:ydata[idx]})
        writer.add_summary(summary, i)
    a_value,b_value = sess.run([a,b])
    print(a_value)
    print(b_value)
```
以上为所有代码，具体分析代码，顺便分析tensorflow构建流的流程，首先我们需要定义变量x和y，在tensorflow中我们用占位符来定义两个变量，这两个变量可以不用进行任何操作，接下来定义我们的参变量a和b，在变量都定义好了之后我们就可以开始我们的模型构建了，也就是我们的op，这个op很简单就是$y=a\cdot x+b$,构建这个op之后我们需要定义损失函数，损失函数在我们这个应用中是很简单的，实际上在所有应用中损失函数都比较简单，就是真实值与目标值之间的差值的平方最小，或者是平方和最小，有了这个损失函数的定义，我们就能很快的写出损失函数 $loss = (y_-y)^2$我们在这里采用L2范损失函数的定义，最后需要利用梯度下降算法求损失函数的极小值，我们就简单使用梯度下降算法了，实际上tensorflow提供了多种自动求导的算法，在这里就不对算法进行详细说明了，得到这些东西后我们下面就是获取真正的数据xdata和ydata了，训练数据获取的方式很多，从影像，从文件或者自定义，这些都不是问题，此外我们需要面对的主要问题在于如何让我们构建的图run起来，实际上run是通过session来实现的，从代码中我们可以看到，首先我们对变量进行了初始化，实际上需要进行初始化的变量只有a和b，然后我们循环2000次每次都从xdata和ydata中随机取数据进行计算，然后得到计算的结果为a和b。我们输出的图为：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/graph_run%3Dsimple-logististic.png" align=center>  
从图中可以对我们的流程进行分析，当然实际上图要更细致的分析，而且输入参数，loss变量都可以通过tensorboard进行分析，再这里我们并不对此进行详细分析。通过这一次的实验更加深入的理解了tensorflow的图的机制，同时对session的run也有了更加深刻的理解。
