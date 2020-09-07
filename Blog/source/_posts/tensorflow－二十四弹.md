---
title: tensorflow－二十四弹
date: 2018-02-28 22:27:13
tags: tensorflow学习
categories: 学习
---
在上次的学习中我学习了如何读取模型，现在我们已经了解了如何训练和如何读取模型，于是我就想做一些真正的应用，如何才能将机器学习应用起来，或者说如何给别人用，这个是需要思考的问题，结合前一段时间学习的通过python搭建服务器，我想到，是不是可以将我们的深度学习的模型也打一个包然后通过web服务的形式给用户使用呢？我觉得这是一个好想法，因此我开始着手进行工作．抛开服务器部分不谈，我们先来谈谈学习过程这个部分.  
为了能够将模型进行更好的模块化，参考网上资料，我将整个过程分为三个模块，分别为：
* １．网络定义模块
* ２．模型训练模块
* ３．识别预测模块

根据三个模块的名称我们能够很方便的了解各个模块的作用，为了帮助理解，我说明一下为什么要分为三个模块而不是简单的分为模型训练和预测识别两个模块，实际上我们在训练识别过程中都是同样的神经网络模型，如果不将网络定义模块独立出来，则可能出现网络重复定义的问题，而且随着网络变得越来越复杂，两次网络定义出现不一致的可能性就越大，因此需要将网络模型的定义独立出来，在训练和识别的过程中进行调用；首先看网络定义模块：
```python
#!/usr/bin/env python
#-*- coding:utf-8 -*-
import tensorflow as tf


class Network:
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

    def __init__(self):
        self.learning_rate = 0.001
        # 记录已经训练的次数
        self.global_step = tf.Variable(0, trainable=False)

        self.x = tf.placeholder(tf.float32, [None, 784])
        self.label = tf.placeholder(tf.float32, [None, 10])
        self.x_image = tf.reshape(self.x, [-1,28,28,1])

        #第一层
        self.w1 = self.weight_variable([5,5,1,32])#5×5的卷积核 32种特征
        self.b1 = self.bias_variable([32])
        self.h1 = tf.nn.relu(self.conv2d(self.x_image, self.w1) + self.b1)
        self.h_pool1 = self.max_pool_2x2(self.h1) #池化

        #第二层
        self.w2 = self.weight_variable([5,5,32,64])
        self.b2 = self.bias_variable([64])
        self.h_conv2 = tf.nn.relu(self.conv2d(self.h_pool1, self.w2) + self.b2)
        self.h_pool2 = self.max_pool_2x2(self.h_conv2)

        #全连接
        self.W_fc1 = self.weight_variable([7*7*64, 1024])
        self.b_fc1 = self.bias_variable([1024])
        self.h_pool2_flat = tf.reshape(self.h_pool2, [-1, 7*7*64])
        self.h_fc1 = tf.nn.relu(tf.matmul(self.h_pool2_flat, self.W_fc1) + self.b_fc1)

        #输出
        #keep_prob=0.5
        #self.h_fc1_drop = tf.nn.dropout(self.h_fc1, keep_prob)
        self.W_fc2 = self.weight_variable([1024, 10])
        self.b_fc2 = self.bias_variable([10])
        self.y = tf.nn.softmax(tf.matmul(self.h_fc1, self.W_fc2) + self.b_fc2)

        #loss
        self.loss = -tf.reduce_sum(self.label * tf.log(self.y + 1e-10))

        # minimize 可传入参数 global_step， 每次训练 global_step的值会增加1
        # 因此，可以通过计算self.global_step这个张量的值，知道当前训练了多少步
        self.train = tf.train.AdamOptimizer(self.learning_rate).minimize(
            self.loss, global_step=self.global_step)

        predict = tf.equal(tf.argmax(self.label, 1), tf.argmax(self.y, 1))
        self.accuracy = tf.reduce_mean(tf.cast(predict, "float"))
```
从代码中可以看出，实际上就是很简单的一个CNN的神经网楼，包含两个卷积层，一个全连接层和一个输出层，在网络定义中定义了训练方式，损失函数，预测函数已经精度，对于有tensorflow基础的同学来说应该相当简单；在定义好网络之后我们就需要添加模型训练的代码了：
```python
#!/usr/bin/env python
#-*- coding:utf-8 -*-
import tensorflow as tf
import numpy as np

from tensorflow.examples.tutorials.mnist import input_data
mnist = input_data.read_data_sets("../mnist/MNIST_data/",one_hot=True)
from cnnNetwork import Network

class Train:
    def __init__(self):
        self.net = Network()
        self.sess = tf.Session()
        self.sess.run(tf.global_variables_initializer())
        self.data = mnist

    def train(self):
        batch_size = 50
        train_step = 2000
        # 记录训练次数, 初始化为0
        step = 0

        # 每隔1000步保存模型
        save_interval = 1000

        # tf.train.Saver是用来保存训练结果的。
        # max_to_keep 用来设置最多保存多少个模型，默认是5
        # 如果保存的模型超过这个值，最旧的模型将被删除
        saver = tf.train.Saver(max_to_keep=10)

        ckpt = tf.train.get_checkpoint_state('./')
        if ckpt and ckpt.model_checkpoint_path:
            saver.restore(self.sess, ckpt.model_checkpoint_path)
            # 读取网络中的global_step的值，即当前已经训练的次数
            step = self.sess.run(self.net.global_step)
            print('Continue from')
            print('        -> Minibatch update : ', step)

        while step < train_step:
            x, label = self.data.train.next_batch(batch_size)
            _, loss = self.sess.run([self.net.train, self.net.loss],
                                    feed_dict={self.net.x: x, self.net.label: label})
            step = self.sess.run(self.net.global_step)
            if step % 100 == 0:
                print('第%5d步，当前loss：%.2f' % (step, loss))

            # 模型保存在ckpt文件夹下
            # 模型文件名最后会增加global_step的值，比如1000的模型文件名为 model-1000
            #if step % save_interval == 0:
        saver.save(self.sess, 'model', global_step=step)

    def calculate_accuracy(self):
        test_x = self.data.test.images
        test_label = self.data.test.labels
        accuracy = self.sess.run(self.net.accuracy,
                                 feed_dict={self.net.x: test_x, self.net.label: test_label})
        print("准确率: %.2f，共测试了%d张图片 " % (accuracy, len(test_label)))


if __name__ == "__main__":
    app = Train()
    app.train()
    app.calculate_accuracy()

```
额，实际上也没有什么好说的，主要有几点要注意，首先是定义训练的步长，第二点是在训练过程中如果可以尽量保存训练中间步骤，这样可以避免在长时间的训练过程中突然出现异常，则可以从上一次保存的训练参数开始重新训练以节省时间，在以上代码中并没有保存训练结果，不过保存训练结果是有必要，所以在以后训练大型网络模型的时候还是要尽量隔一定时间或者一定步长后保存训练结果，至于训练结果的保存方法，我们在前一次已经详细讨论过，在这里就不进行讨论了；在训练过程中每次训练的数据集大小和训练步长都可以自己设定，这个就没有什么好说的，只是将以前CNN的过程拆分而已；最后的部分就是预测部分，这一部分比较重要，实际上如果要作为服务调用，我们主要也是调用预测这一模块，因此我会比较详细的分析预测部分的代码：
```python
from tensorflow.examples.tutorials.mnist import input_data
mnist = input_data.read_data_sets("../mnist/MNIST_data/",one_hot = True)

import tensorflow as tf
import numpy as np

from cnnNetwork import Network

class Predict:
    def __init__(self):
        self.net = Network()
        self.sess = tf.Session()
        self.sess.run(tf.global_variables_initializer())
        self.data = mnist
        # 加载模型到sess中
        self.restore()

    def restore(self):
        saver = tf.train.Saver()
        ckpt = tf.train.get_checkpoint_state('./')
        if ckpt and ckpt.model_checkpoint_path:
            saver.restore(self.sess, ckpt.model_checkpoint_path)
        else:
            raise FileNotFoundError("未保存任何模型")

    def predict(self, image_path):
        # 读图片并转为黑白的
        x, label = self.data.train.next_batch(10)
        y = self.sess.run(self.net.y, feed_dict={self.net.x: x})

        # 因为x只传入了一张图片，取y[0]即可
        # np.argmax()取得独热编码最大值的下标，即代表的数字
        print('        -> Predict digit', np.argmax(y[0]))
        print('        -> Predict digit', np.argmax(y[1]))
        print('        -> Predict digit', np.argmax(y[2]))
        print('        -> Predict digit', np.argmax(y[3]))
        print('        -> Predict digit', np.argmax(y[4]))
        print('        -> Predict digit', np.argmax(y[5]))

        print(label)

if __name__ == "__main__":
    app = Predict()
    app.predict('./')
```
实际在应用过程中应该是读取影像进行预测，不过为了检测模型我这里直接还是用MNIST数据集进行预测，预测部分主要有两个模块第一个模块是读取模型的模块，实际上也很简单就是读取模型，不过与上一次不同的是，由于这一次我们将网络的定义独立了，因此我们可以直接读取模型进行计算；第二个部分就是预测部分，我们看到，实际上我们在预测过程中通过sess的run函数传递了两个参数，一个是net中的ｙ，这个参数是没有值的，一个是net中的ｘ，这个参数是通过MNIST数据集进行初始化的，理论上来说我们的输入只需要一个x为什么还需要y？我理解是告诉tensorflow输出和输出参数，y是预测结果，而ｘ是待预测的数据，因此需要输入两个参数．  
由此整个训练和预测就结束了，下一节我们会讲讲如何通过python搭建简单的数据库，给出RESTFUL风格的API接口进行预测，另外模型预测部分的代码也会进行适当的修改以适应图像识别的需求．
