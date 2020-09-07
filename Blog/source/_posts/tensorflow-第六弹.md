---
title: tensorflow-第六弹
date: 2017-07-30 22:21:01
tags: tensorflow学习
categories: 学习
mathjax: true
---
好了，经过前很长时间的准备和学习，我们目前应该掌握的技能应该包括
* 1.tensorflow的工作流程，也就是图的构建的流程
* 2.tensorflow的可视化工具，tensorboard的使用以及变量的可视化方法，以及如果查看数据流图
* 3.通过爬虫从网上获取大量数据的方法

（什么？你说你一个都不会，那么请不要看下面的内容了，请先进行基础知识的学习，亲～）  
好了，假设我们有了以上所说的各种技能作为基础，那么我们要做的就是下一步就是进行深度学习框架的构建了，再这里我们首先只是简单的介绍CNN，也就是深度卷积网络的构建，我们将训练和构建一个简单的三层卷积神经网络，当然咯，具体构建多少层那就是根据样本和需要进行研究的事情了，我们在这里只展示三层卷积神经网络的构建作为演示。  
我们在下一掌将讲解整个卷积圣经网络的数学基础，现在我们只是简单的分析一下CNN的构造过程，CNN的构造包括卷积，池化，全连接和输出三个主要的部分，其中卷积池化层可以有多个，因此我们构造的三层神经网络包含两个卷积池化层以及一个全连接层和一个输出层，由于不对其数学原理进行分析，我们直接看看代码和输出的图：  
```python
import tensorflow as tf
import numpy as np

from tensorflow.examples.tutorials.mnist import input_data
mnist = input_data.read_data_sets("mnist/MNIST_data/",one_hot=True)

#input
with tf.name_scope('input'):
    x = tf.placeholder(tf.float32, [None, 784],name = 'x-input')
    y = tf.placeholder("float", [None, 10],name='y-input')
    x_image = tf.reshape(x,[-1,28,28,1],name='img')
    tf.summary.image('image',x_image,20)

#layer1
with tf.name_scope('layer1'):
    w1 = tf.Variable(tf.truncated_normal([5,5,1,32],stddev=0.1))    #initial size=[5*5*32] 32 filter features
    b1 = tf.Variable(tf.constant(0.1,shape=[32]))
    #conv & relu
    out1 = tf.nn.relu(tf.nn.conv2d(x_image,w1,strides=[1,1,1,1],padding='SAME')+b1)
    #pool
    out_pool1 = tf.nn.max_pool(out1,ksize=[1,2,2,1],strides=[1,2,2,1],padding='SAME')

    tf.summary.histogram('layer1/w1',w1)
    tf.summary.histogram('layer1/b1',b1)

#layer2
with tf.name_scope('layer2'):
    w2 = tf.Variable(tf.truncated_normal([5,5,32,64],stddev=0.1))
    b2 = b1 = tf.Variable(tf.constant(0.1,shape=[64]))
    #conv & relu
    out2 = tf.nn.relu(tf.nn.conv2d(out_pool1,w2,strides=[1,1,1,1],padding='SAME')+b2)
    #pool
    out_pool2 = tf.nn.max_pool(out2,ksize=[1,2,2,1],strides=[1,2,2,1],padding='SAME')
    tf.summary.histogram('layer2/w2',w2)
    tf.summary.histogram('layer2/b2',b2)

#full-connected
with tf.name_scope('full_connected'):
    #28/2/2
    wf = tf.Variable(tf.truncated_normal([7*7*64,1024],stddev=0.1))
    bf = tf.Variable(tf.constant(0.1,shape=[1024]))
    out2t = tf.reshape(out_pool2, [-1, 7*7*64])#trans from 2D to 1D
    outf = tf.nn.relu(tf.matmul(out2t,wf)+bf)

#output
with tf.name_scope('output'):
    keep_prob = tf.placeholder("float") #point exist over probility
    drop = tf.nn.dropout(outf, keep_prob)
    wo = tf.Variable(tf.truncated_normal([1024,10],stddev=0.1))
    bo = tf.Variable(tf.constant(0.1,shape=[10]))
    outy = tf.nn.softmax(tf.matmul(drop,wo)+bo)

with tf.name_scope('corss_ectropy'):
    cross_entropy = -tf.reduce_sum(y*tf.log(outy))
    tf.summary.scalar('cross_entropy',cross_entropy)

train_step = tf.train.AdamOptimizer(1e-4).minimize(cross_entropy)
correct_predict = tf.equal(tf.argmax(outy, 1), tf.argmax(y, 1))
accuracy = tf.reduce_mean(tf.cast(correct_predict, "float"))
merged = tf.summary.merge_all()

with tf.Session() as sess:
    writer = tf.summary.FileWriter('tensorflow-start_Log', sess.graph)
    sess.run(tf.global_variables_initializer())
    for i in range(2000):
        batch = mnist.train.next_batch(50)
        if i%100 == 0:
            train_acc = accuracy.eval(feed_dict={x:batch[0], y: batch[1], keep_prob: 1.0})
            #summary,_ =sess.run([merged,train_step],feed_dict={x:batch[0], y_:batch[1], keep_prob:1.0})
            print('step',i,'training accuracy',train_acc)
        summary,_ = sess.run([merged,train_step],feed_dict={x:batch[0], y:batch[1], keep_prob:0.5})
        writer.add_summary(summary, i)
```
代码看起来有点长，其实其结构很清晰，都通过with模块进行区分了，同样我们首先是构建计算图，从输入开始，输入数据包括x，y，x为需要进行识别的影像，而y为影像的label，由于进行卷积操作，所以将图像转换为28×28的二维影像矩阵了，然后构建第一个卷积层，包括卷积函数的权重，卷积核的个数以及偏置变量，然后进行卷积操作，在这里tensorflow提供了一套方便的函数进行卷积，然后得到一个输出，为了减小运算量，我们需要对输出进行池化，池化的过程我们在下一章的数学基础上详细描述，池化完成后28×28的影像大小就变为14*14,然后继续进行第二层的卷积和池化操作，进行完卷积和池化层的操作后就面临着全连接，此时就相当于一个BP网络，其输入为两次卷积后的输出，我们这里为一个7×7×64个节点的数据，然后中间层我们在这里设置为1024个节点。实际上节点个数可以自定义，最后到输出层与label数据进行比较，通过交叉熵来评价精度，然后通过Adam算法进行迭代解算，到此为止我们算已经构建完成所有的图了，然后接下来的工作就是启动运算，启动运算的方法与前面介绍的方法差不多也就不进行详细介绍了，到此我们已经完成了构建一个三层的深度神经网络的构建，我们看看计算图和计算精度：
![enter image description here](https://lh3.googleusercontent.com/xVJLeFGTuex5kZZYIfjVXWeBSbaRNUYgTUVTKOCMbEiLmjfNW0F16SJpubH8tZNLzvzlZOyX_U8r=s0 "cnnMnistGraph.png")
从计算图中我们可以看到整个从输入到输出的数据流的全过程，每个节点可以展开进行更细致的查看
![enter image description here](https://lh3.googleusercontent.com/IR0UIJ87vSb7sdRXENtz-WX8uyIFtIzXU0n8TbzeNXaRk8dDW6D6F3gTFlM-flZqULKSbk0doup8=s0 "cnnMnistCrossEctropy.png")
上图为误差熵的变化，我们可以看到其值随着训练次数的增加具有明显的减小。以上就是搭建一个三层的卷积神经网络的全过程，下一讲会详细描述深度神经网络的数学原理，结合我们今天的内容可以对CNN有一个更加深刻的理解。
