---
title: tensorflow-二十六弹
date: 2018-08-13 22:54:33
tags: tensorflow学习
categories: 学习
---
&nbsp;&nbsp;&nbsp;&nbsp;写了很多关于tensorflow的部分，但是都比较零散，因为以前不管是对于python还是对于机器学习都了解得不够深刻，因此写出来的东西也就显得比较零散，代码不能够构成一个体系，所以也就谈不上什么积累，不过是多了解了一些关于tensorflow的东西而已，现在不管是对于python还是对于深度学习都有了更加深刻的理解，所以准备重新对整个过程进行组织，代码进行更加有条理的重构，以便于进行进一步的扩展。  
&nbsp;&nbsp;目前只搭了一个CNN的框架，整个构架描述为：
  * 1.定义神经网络的基本结构单元：
  
&nbsp;&nbsp;&nbsp;&nbsp;实际上神经所有的神经网络都会有一些基本的结构体，如权重，如偏置；因为就目前来说神经网络就是n多的拟合线性运算加上一个响应函数进行拟合，所以基本的结构单元都是相似的，因此我们定义了一个神经网络基类来初始化这些基础的结构
  ```python
  #基础网络功能，包括：
#1.权重定义
#2.增益的定义
#3.二维卷积运算
#4.最大值池化
class BaseNet:
    #初始化权重
    def weight_variable(self,shape):
        #从截断的正态分布中输出随机值
        initial = tf.truncated_normal(shape, stddev=0.1)
        return tf.Variable(initial)

    #初始化偏置
    def bias_variable(self,shape):
        #设置常数为0.1
        initial = tf.constant(0.1, shape=shape)
        return tf.Variable(initial)

    #二维卷积运算
    def conv2d(self,x, W):
        return tf.nn.conv2d(x, W, strides=[1,1,1,1], padding="SAME")

    #最大值池化
    def max_pool_2x2(self,x):
        return tf.nn.max_pool(x, ksize=[1,2,2,1],
                              strides=[1,2,2,1], padding="SAME")
  ```
&nbsp;&nbsp;&nbsp;&nbsp;从上面的代码中可以看到，我们定义的结构体包括：1.权重变量的定义；2.偏置变量的定义；3.卷积运算；4.池化操作；实际上卷积运算不是所有神经网络通用的操作，仅仅是卷积神经网络需要的操作，但是我们目前就是处理卷积神经网络，为了方便就这么写了。  
* 2.特殊网络结构的定义：
  
&nbsp;&nbsp;&nbsp;&nbsp;定义好了基本的神经网络结构以后剩下的工作就是针对某一个神经网络进行特殊的定义，目前我们只定义了卷积神经网络，实际上卷神经网络相对比较简单，主要的结构有两个部分，第一个是卷积层，通过卷积层可以进行参数共享从而减小参数个数，另外通过不同的卷积核实际上可以提取不同的特征从而对目标进行识别，另外一个是池化操作，池化操作是神经网络的一个巨大创新，通过池化这个简单的操作对近邻的特征进行概括，并且通过池化操作可以得到待识别目标的旋转不变特征。
* 3. 根据数据对网络进行实例化：

```python
#!/usr/bin/env python
#-*- coding:utf-8 -*-

import numpy as np
import tensorflow as tf

from model import LeNet
from tensorflow.examples.tutorials.mnist import input_data
mnist = input_data.read_data_sets("../data/MNIST_data/",one_hot=True)

ckptfiles = './mnist_LeNet_ckpt/'

class mnistLeNet(LeNet):
    #初始化网络
    def __init__(self):
        LeNet.__init__(self,28,28)

        #控制变量定义
        self.learning_rate = 0.001
        # 记录已经训练的次数
        self.global_step = tf.Variable(0, trainable=False)
        self.x = tf.placeholder(tf.float32, [None, 784])
        self.label = tf.placeholder(tf.float32, [None, 10])
        self.x_image = tf.reshape(self.x, [-1,28,28,1])
        
        #网路层次定义
        self.layer1(5,5,32)
        self.layer2(5,5,64)
        self.fullConnLayer(int(1024))
        self.outputLayer(10)

        #计算参数定义
        #loss
        self.loss = -tf.reduce_sum(self.label * tf.log(self.y + 1e-10))

        # minimize 可传入参数 global_step， 每次训练 global_step的值会增加1
        # 因此，可以通过计算self.global_step这个张量的值，知道当前训练了多少步
        self.train = tf.train.AdamOptimizer(self.learning_rate).minimize(
            self.loss, global_step=self.global_step)

        predict = tf.equal(tf.argmax(self.label, 1), tf.argmax(self.y, 1))
        self.accuracy = tf.reduce_mean(tf.cast(predict, "float"))

class TrainMnistLeNet:
    def __init__(self):
        self.net = mnistLeNet()
        self.sess = tf.Session()
        self.sess.run(tf.global_variables_initializer())
        self.data = mnist

    #模型训练过程
    def trainMnist(self):
        batch_size = 50
        train_step = 3000
        # 记录训练次数, 初始化为0
        step = 0
        save_interval = 1000

        batch_size = 50
        train_step = 3000
        # 记录训练次数, 初始化为0
        step = 0

        # 每隔1000步保存模型
        #save_interval = 1000

        # tf.train.Saver是用来保存训练结果的。
        # max_to_keep 用来设置最多保存多少个模型，默认是5
        # 如果保存的模型超过这个值，最旧的模型将被删除
        saver = tf.train.Saver(max_to_keep=10)
        ckpt  = tf.train.get_checkpoint_state(ckptfiles)
        merged = tf.summary.merge_all()
        writer = tf.summary.FileWriter(ckptfiles+'graph',self.sess.graph)

        if ckpt and ckpt.model_checkpoint_path:
            saver.restore(self.sess, ckpt.model_checkpoint_path)
            # 读取网络中的global_step的值，即当前已经训练的次数
            step = self.sess.run(self.global_step)
            print('Continue from')
            print('        -> Minibatch update : ', step)

        while step < train_step:
            x, label = self.data.train.next_batch(batch_size)
            _, loss = self.sess.run([self.net.train, self.net.loss],
                                    feed_dict={self.net.x: x, self.net.label: label})
            step = self.sess.run(self.net.global_step)
            rs=self.sess.run(merged)
            writer.add_summary(rs, step)
            if step % 100 == 0:
                print('第%5d步，当前loss：%.2f' % (step, loss))

        # 模型保存在ckpt文件夹下
        # 模型文件名最后会增加global_step的值，比如1000的模型文件名为 model-1000
        #if step % save_interval == 0:
        #只保存一次模型
        saver.save(self.sess, ckptfiles+'model', global_step=step)

    def calculate_accuracy(self):
        test_x = self.data.test.images
        test_label = self.data.test.labels
        accuracy = self.sess.run(self.net.accuracy,
                                 feed_dict={self.net.x: test_x, self.net.label: test_label})
        print("准确率: %.2f，共测试了%d张图片 " % (accuracy, len(test_label)))

if __name__ == "__main__":
    app = TrainMnistLeNet()
    app.trainMnist()
    app.calculate_accuracy()
```
&nbsp;&nbsp;&nbsp;&nbsp;这个步骤其实比较简单，就是根据数据对整个网络的输入和输出进行实例化操作，在我的代码中是训练的Mnist数据，所以定义输入的变量的28*28的数据，结果为10个数字，然后获取数据进行训练，由于Mnist数据的解析和处理都在tensorflow的example中被处理了，在处理过程中就省略了这个过程，实际上应该定义一个类专门对数据进行处理和优化。