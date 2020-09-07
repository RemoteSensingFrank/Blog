---
title: tensorflow-三十弹
date: 2020-03-03 09:06:38
tags: tensorflow学习
categories: 学习
mathjax: true
---
&nbsp;&nbsp;&nbsp;&nbsp;在上一部分中我们只描述了深度学习的一些基本问题，如梯度消失，激活函数等问题，剩下两个部分，分别为tensorflow2.x的使用以及关于GAN的一些知识与实践，首先介绍一下关于tensorflow2.x 以及其集成的kersa的知识，以前的时候写神经网络总是通过tensorflow原生直接写，虽然也提供了自动求导方式，但是实践起来总是有点麻烦，而使用kersa之后感觉惊为天人，网络的构建过程得到了极大的简化，使得构建深度学习模型变成了一个工程问题而非科学问题了。
&nbsp;&nbsp;&nbsp;&nbsp;下面我们首先介绍一下**tensorflow2.0**，首先安装过程，目前来说tensorflow2.0的安装不支持python3.7，我在python3.6 ubuntu18.04 LST版本下安装，我本机系统式win10，采用WSL的方式安装了linux的控制台环境，反正目前也不需要图形化环境，所以采用此种方式最为简单方便，安装在linux下安装tensorflow2.0的环境相对还是比较简单，在连接网络的条件下[参考](https://zhuanlan.zhihu.com/p/61472293)进行就可以了。安装完成之后我们就可以开始进行tensorflow代码的编写了，原生的tensorflow代码实际上区别不大，我们主要介绍一下关于kersa代码的一些特点：
```python
model = tf.keras.Sequential()
model.add(layers.Conv2D(8,(5,5), activation='relu',input_shape=(28,28,1)))
model.add(layers.Conv2D(16,(5,5),activation='relu',input_shape=(28,28,1)))
model.add(layers.MaxPool2D(pool_size=(2,2)))
model.add(layers.Flatten())
model.add(layers.Dense(128, activation='relu'))#784向量映射到128
model.add(layers.Dropout(0.2))  #dropout trick 避免模型过于依赖某些神经元避免在分类过程中
model.add(layers.Dense(10, activation='softmax'))
model.compile(optimizer='adam',
                loss='sparse_categorical_crossentropy',
                metrics=['accuracy'])
```
以上代码构建了一个简单的深度神经网络的学习模型，从以上代码中有几个函数，我们进行详细说明：
* Sequential 构建一个序贯模型：序贯模型是相对于循环网络来说的，输入经过处理到输出，而不需要循环输入的网络模型就成为序贯模型；
* Conv2D：卷积层，这个是CNN网络的典型结构，对网络做一个卷积操作，卷积层包括卷积核大小以及卷积特征数；
* MaxPool2D：池化层，实际上池化操作在以前的部分也有，就是一个求池化核大小内的最大值的区别；
* Flatten：压平，将一个n维向量转换为一维的过程；
* Dense：全连接层
* compile：实际上这个就是定义网络的目标函数，损失函数以及优化方法的定义，通过compile构成一个能够反向传播的网络，在compile之后就可以进行数据优化的训练过程了；

&nbsp;&nbsp;&nbsp;&nbsp;从上面一小段代码可以看出，通过kersa将神经网简单的分成了几个部分，包括：数据准备（数据读取和数据格式转换），模型构建（根据数学原理和神经网络结构构建模型），模型训练。为了能够更加清楚的说明这个过程，编写了一个简单的网络模型作为参考：
```python
'''
@Descripttion: tensorflow 2.0 kersa学习第一课
@version: 1.0版本
@Author: Frank.Wu
@Date: 2020-02-26 09:50:02
@LastEditors: Frank.Wu
@LastEditTime: 2020-03-03 11:33:48
'''
#coding = utf-8

import tensorflow as tf
import numpy as np
from tensorflow import keras
from tensorflow.keras import layers, optimizers, Sequential,datasets
import  os


print(tf.__version__)
print(tf.keras.__version__)

# print(os.environ.keys())
# 设置后台打印日志等级 避免后台打印一些无用的信息
os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'

''' 
    加载mnist数据，判断本地是否有数据，
    如果有则加载，否则加载在线数据
'''
def load_mnist_data():
    pathLocal = './data/mnist/mnist.npz'
    if(os.path.exists(pathLocal)):
        f = np.load(pathLocal)
        x_train, y_train = f['x_train'], f['y_train']
        x_test, y_test = f['x_test'], f['y_test']
        f.close()
        return (x_train, y_train), (x_test, y_test)
    else:
        return datasets.mnist.load_data()


'''
    构建模型:
'''
#1.mnist 全连接模型
def mnist_full_model():
    model = tf.keras.Sequential()
    model.add(layers.Flatten(input_shape=(28, 28)))#将28*28 转换到一维
    model.add(layers.Dense(128, activation='relu'))#784向量映射到128
    model.add(layers.Dropout(0.2))  #dropout trick 避免模型过于依赖某些神经元避免在分类过程中
    model.add(layers.Dense(10, activation='softmax'))
    model.compile(optimizer='adam',
                  loss='sparse_categorical_crossentropy',
                  metrics=['accuracy'])
    #将模型输出
    keras.utils.plot_model(model, to_file='./data/1_model.png', show_shapes=True)
    #plot(model, to_file='./data/1_model.png', show_shapes=True)
    return model

def mnist_cnn_model():
    model = tf.keras.Sequential()
    model.add(layers.Conv2D(8,(5,5), activation='relu',input_shape=(28,28,1)))
    model.add(layers.Conv2D(16,(5,5),activation='relu',input_shape=(28,28,1)))
    model.add(layers.MaxPool2D(pool_size=(2,2)))
    model.add(layers.Flatten())
    model.add(layers.Dense(128, activation='relu'))#784向量映射到128
    model.add(layers.Dropout(0.2))  #dropout trick 避免模型过于依赖某些神经元避免在分类过程中
    model.add(layers.Dense(10, activation='softmax'))
    model.compile(optimizer='adam',
                  loss='sparse_categorical_crossentropy',
                  metrics=['accuracy'])
    #将模型输出
    keras.utils.plot_model(model, to_file='./data/1_model.png', show_shapes=True)
    #plot(model, to_file='./data/1_model.png', show_shapes=True)
    return model
 
'''
    通过数据进行训练
'''
#1.mnist 数据集进行训练
def mnist_fit_evaluate(model,x_train,y_train,x_test,y_test):
    model.fit(x_train, y_train,batch_size=64, epochs=5)
    model.evaluate(x_test,  y_test, verbose=2)
    model.save('./data/1_mnist.h5')
    return model



'''
    全流程
'''
def mnist_full():
    pathModel = './data/1_mnist.h5'

    if(os.path.exists(pathModel)):
        model = keras.models.load_model(pathModel)
        return model
    else:
        model = mnist_cnn_model()
        (x_train, y_train), (x_test, y_test) = load_mnist_data()
        x_train, x_test=x_train/255.0, x_test/255.0
        x_train = x_train.reshape(60000, 28, 28, 1)
        x_test = x_test.reshape(10000, 28, 28, 1)
        mnist_fit_evaluate(model,x_train,y_train,x_test,y_test)
        return model
        

mnist_full()
```
以上代码数据采用的是mnist数据，因为提供的mnist数据加载比较慢，所以我们从网上下载了数据并对数据读取代码进行了略微的修改。数据读取完成后得到训练样本集和测试样本集，训练样本集模型层编写了两种模型，包括简单的神经网络模型以及深度神经网络模型，对两种模型进行了训练，并查看训练结果的准确性。另外keras还提供了将模型结构输出的函数，可以将模型结构保存为图片。如果大家有兴趣可以在tensorflow2.0环境下运行以上代码进行测试，以便进一步了解keras的数据模型。