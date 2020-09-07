---
title: tensorflow-二十三弹
date: 2018-02-25 20:54:40
tags: tensorflow学习
categories: 学习
---
今天就tensorflow训练好的模型的存取问题进行一些讨论，主要翻译了一下一篇英文博客，另外加上了一些自己的尝试和处理经验，英文博客的地址为：http://cv-tricks.com/tensorflow-tutorial/save-restore-tensorflow-models-quick-complete-tutorial/　这个是官方的一个说明，我先翻译这个文章，如果英文好的可以直接跳过，主要内容为：
文章主要分为四个部分，第一个部分的介绍我们不进行翻译，相信对tensorflow有基本了解的同学都能懂，另外前几篇博客中都有介绍，我直接从第二个部分开始翻译：

## ２．tensorflow模型的保存
我们说如果对于一个图像分类的应用，训练了一个卷积神经网络．在训练的过程中我们可以查看loss和accuracy变量，一旦训练完成则可以手动停止训练过程，或者参数不再变化后停止训练过程．当训练完成后需要将训练结果保存起来以便于以后进行应用，而在tensorflow中如果你希望保存整个网络图和所有变量，我们需要创建一个tf.train.Saver()类的实例，如下：  
saver = tf.train.Saver()  
记住，tensorflow的变量只有在session中才被激活，因此如果希望保存图，则必须要在session中调用创建的saver变量
```python
saver = tf.train.Saver()
saver.save(sess, 'my-test-model')
```
在这里，sess是session的一个对象，‘my-test-model’是你想保存模型的文件名，下面是完整的代码示例：
```python
import tensorflow as tf
w1 = tf.Variable(tf.random_normal(shape=[2]), name='w1')
w2 = tf.Variable(tf.random_normal(shape=[5]), name='w2')
saver = tf.train.Saver()
sess = tf.Session()
sess.run(tf.global_variables_initializer())
saver.save(sess, 'my_test_model')

# This will save following files in Tensorflow v >= 0.11
# my_test_model.data-00000-of-00001
# my_test_model.index
# my_test_model.meta
# checkpoint
```

 ```python
import tensorflow as tf
w1 = tf.Variable(tf.random_normal(shape=[2]), name='w1')
w2 = tf.Variable(tf.random_normal(shape=[5]), name='w2')
saver = tf.train.Saver()
sess = tf.Session()
sess.run(tf.global_variables_initializer())
saver.save(sess, 'my_test_model')

# This will save following files in Tensorflow v >= 0.11
# my_test_model.data-00000-of-00001
# my_test_model.index
# my_test_model.meta
# checkpoint
```
如果我们想在模型训练一千次后将模型保存，我们可以通过如下代码实现：  
saver.save(sess, 'my_test_model',global_step=1000)  
以上代码会在模型名称后添加‘-1000’的后缀，另外如下文件将会被创建：

my_test_model-1000.index  
my_test_model-1000.meta  
my_test_model-1000.data-00000-of-00001  
checkpoint  


my_test_model-1000.index  
my_test_model-1000.meta  
my_test_model-1000.data-00000-of-00001  
checkpoint  


在训练过程中我们希望每迭代1000次就保存一次模型，但是.meta文件只会在最开始被创建一次，我们不需要多次创建，我们只需要将最近一次的迭代参数写入文件中，当我们不想将训练参数写入文件中时我们可以通过如下代码实现：
```python
saver.save(sess, 'my-model', global_step=step,write_meta_graph=False)


saver.save(sess, 'my-model', global_step=step,write_meta_graph=False)
```
If you want to keep only 4 latest models and want to save one model after every 2 hours during training you can use max_to_keep and keep_checkpoint_every_n_hours like this.

```python
#saves a model every 2 hours and maximum 4 latest models are saved.
saver = tf.train.Saver(max_to_keep=4, keep_checkpoint_every_n_hours=2)
```

 ```python
#saves a model every 2 hours and maximum 4 latest models are saved.
saver = tf.train.Saver(max_to_keep=4, keep_checkpoint_every_n_hours=2)
```
注意，我们在保存过程中没有对任何变量进行指定，在这样的情况下会保存所有变量，如果我们不想保存所有变量，我们可以在创建实例的时候指出哪些变量是需要保存的，示例代码如下：
```python
import tensorflow as tf
w1 = tf.Variable(tf.random_normal(shape=[2]), name='w1')
w2 = tf.Variable(tf.random_normal(shape=[5]), name='w2')
saver = tf.train.Saver([w1,w2])
sess = tf.Session()
sess.run(tf.global_variables_initializer())
saver.save(sess, 'my_test_model',global_step=1000)
```

 ```python
import tensorflow as tf
w1 = tf.Variable(tf.random_normal(shape=[2]), name='w1')
w2 = tf.Variable(tf.random_normal(shape=[5]), name='w2')
saver = tf.train.Saver([w1,w2])
sess = tf.Session()
sess.run(tf.global_variables_initializer())
saver.save(sess, 'my_test_model',global_step=1000)
```
通过以上方法可以指定graph中所有部分的变量
## 3. 导入训练好的模型:
如果你想使用他人训练好的模型，你需要做以下两件事情：  
a) 创建网络网络:  
你可以通过编写python代码手动创建与原始模型一样的网络，或者如果你觉得这么做太麻烦，由于我们将网络结构已经保存在meta文件中，因此我们可以直接导入网络模型：  
saver = tf.train.import_meta_graph('my_test_model-1000.meta')  
记住，在.meta文件中只保存了网络结构，因此在导入网络后还需要加载在图中训练的参数

b) 加载模型参数:  
我们在保存网络的过程中也保存了网络参数，因此我们可以通过tf.train.Saver()类的实例来读取神经网络的参数：

```python
with tf.Session() as sess:
  new_saver = tf.train.import_meta_graph('my_test_model-1000.meta')
  new_saver.restore(sess, tf.train.latest_checkpoint('./'))
```
 ```python
with tf.Session() as sess:
  new_saver = tf.train.import_meta_graph('my_test_model-1000.meta')
  new_saver.restore(sess, tf.train.latest_checkpoint('./'))
```
加载参数后我们如果想要获取某一个张量可以通过如下代码实现：
```python
with tf.Session() as sess:    
    saver = tf.train.import_meta_graph('my-model-1000.meta')
    saver.restore(sess,tf.train.latest_checkpoint('./'))
    print(sess.run('w1:0'))
##Model has been restored. Above statement will print the saved value of w1.
```
 ```python
with tf.Session() as sess:    
    saver = tf.train.import_meta_graph('my-model-1000.meta')
    saver.restore(sess,tf.train.latest_checkpoint('./'))
    print(sess.run('w1:0'))
##Model has been restored. Above statement will print the saved value of w1.
```
到目前为止，你已经了解了如何保存和导入tensorflow的模型，在下一章中会详细说明如何去使用和加载训练号的模型

## 4. 加载训练好的模型的实例

现在你应该已经知道如何保存和加载模型了，我们进一步说明如何加载训练好的模型以及如何通过模型进行预测，拟合或通过模型进行进一步的训练．在使用tensorflow训练的过程中定义的计算图中包括训练数据以及一些高维变量如学习率，训练步数等．标准的做法是所有变量都使用placeholders来定义，我们首先创建一个网络然后保存网络，网络保存后placeholders定义的变量的值不会被存储

```python
import tensorflow as tf

#Prepare to feed input, i.e. feed_dict and placeholders
w1 = tf.placeholder("float", name="w1")
w2 = tf.placeholder("float", name="w2")
b1= tf.Variable(2.0,name="bias")
feed_dict ={w1:4,w2:8}

#Define a test operation that we will restore
w3 = tf.add(w1,w2)
w4 = tf.multiply(w3,b1,name="op_to_restore")
sess = tf.Session()
sess.run(tf.global_variables_initializer())

#Create a saver object which will save all the variables
saver = tf.train.Saver()

#Run the operation by feeding input
print sess.run(w4,feed_dict)
#Prints 24 which is sum of (w1+w2)*b1

#Now, save the graph
saver.save(sess, 'my_test_model',global_step=1000)
	```
 ```python
import tensorflow as tf

#Prepare to feed input, i.e. feed_dict and placeholders
w1 = tf.placeholder("float", name="w1")
w2 = tf.placeholder("float", name="w2")
b1= tf.Variable(2.0,name="bias")
feed_dict ={w1:4,w2:8}

#Define a test operation that we will restore
w3 = tf.add(w1,w2)
w4 = tf.multiply(w3,b1,name="op_to_restore")
sess = tf.Session()
sess.run(tf.global_variables_initializer())

#Create a saver object which will save all the variables
saver = tf.train.Saver()

#Run the operation by feeding input
print sess.run(w4,feed_dict)
#Prints 24 which is sum of (w1+w2)*b1

#Now, save the graph
saver.save(sess, 'my_test_model',global_step=1000)
```
现在当我们想要保存模型的时候我们只需要保存图和权重，另外准备一个新的feed_dict就可以使用心得数据训练网络了，另外我们也可以获取到村包的操作和placeholder变量通过graph.get_tensor_by_name()方法
* How to access saved variable/Tensor/placeholders
```python
w1 = graph.get_tensor_by_name("w1:0")
```
* How to access saved operation
```python
op_to_restore = graph.get_tensor_by_name("op_to_restore:0")
```


* How to access saved variable/Tensor/placeholders
```python
w1 = graph.get_tensor_by_name("w1:0")
 ```
* How to access saved operation
```python
op_to_restore = graph.get_tensor_by_name("op_to_restore:0")
```
如果我们只是想对于不同的训练数据使用模型训练，则我们只需要简单的通过feed_dict传递参数就好了

```python
import tensorflow as tf

sess=tf.Session()    
#First let's load meta graph and restore weights
saver = tf.train.import_meta_graph('my_test_model-1000.meta')
saver.restore(sess,tf.train.latest_checkpoint('./'))


# Now, let's access and create placeholders variables and
# create feed-dict to feed new data

graph = tf.get_default_graph()
w1 = graph.get_tensor_by_name("w1:0")
w2 = graph.get_tensor_by_name("w2:0")
feed_dict ={w1:13.0,w2:17.0}

#Now, access the op that you want to run.
op_to_restore = graph.get_tensor_by_name("op_to_restore:0")

print sess.run(op_to_restore,feed_dict)
#This will print 60 which is calculated
#using new values of w1 and w2 and saved value of b1.
```

 ```python
import tensorflow as tf

sess=tf.Session()    
#First let's load meta graph and restore weights
saver = tf.train.import_meta_graph('my_test_model-1000.meta')
saver.restore(sess,tf.train.latest_checkpoint('./'))


# Now, let's access and create placeholders variables and
# create feed-dict to feed new data

graph = tf.get_default_graph()
w1 = graph.get_tensor_by_name("w1:0")
w2 = graph.get_tensor_by_name("w2:0")
feed_dict ={w1:13.0,w2:17.0}

#Now, access the op that you want to run.
op_to_restore = graph.get_tensor_by_name("op_to_restore:0")

print sess.run(op_to_restore,feed_dict)
#This will print 60 which is calculated
#using new values of w1 and w2 and saved value of b1.
```
如果你需要在原始的模型基础上添加更多的操作，添加更多层进行重新训练，则代码如下：
```python
import tensorflow as tf

sess=tf.Session()    
#First let's load meta graph and restore weights
saver = tf.train.import_meta_graph('my_test_model-1000.meta')
saver.restore(sess,tf.train.latest_checkpoint('./'))


* Now, let's access and create placeholders variables and
* create feed-dict to feed new data

graph = tf.get_default_graph()
w1 = graph.get_tensor_by_name("w1:0")
w2 = graph.get_tensor_by_name("w2:0")
feed_dict ={w1:13.0,w2:17.0}

* Now, access the op that you want to run.
op_to_restore = graph.get_tensor_by_name("op_to_restore:0")

* Add more to the current graph
add_on_op = tf.multiply(op_to_restore,2)

print sess.run(add_on_op,feed_dict)
* This will print 120.
```
 ```python
import tensorflow as tf

sess=tf.Session()    
* First let's load meta graph and restore weights
saver = tf.train.import_meta_graph('my_test_model-1000.meta')
saver.restore(sess,tf.train.latest_checkpoint('./'))


* Now, let's access and create placeholders variables and
* create feed-dict to feed new data

graph = tf.get_default_graph()
w1 = graph.get_tensor_by_name("w1:0")
w2 = graph.get_tensor_by_name("w2:0")
feed_dict ={w1:13.0,w2:17.0}

* Now, access the op that you want to run.
op_to_restore = graph.get_tensor_by_name("op_to_restore:0")

* Add more to the current graph
add_on_op = tf.multiply(op_to_restore,2)

print sess.run(add_on_op,feed_dict)
* This will print 120.
```
另外你也可以保存部分就的网络结构并添加新的结构去训练更好的网络
......
......
saver = tf.train.import_meta_graph('vgg.meta')
* Access the graph
graph = tf.get_default_graph()
* Prepare the feed_dict for feeding data for fine-tuning

* Access the appropriate output for fine-tuning
fc7= graph.get_tensor_by_name('fc7:0')

* use this if you only want to change gradients of the last layer
```python
fc7 = tf.stop_gradient(fc7) # It's an identity function
fc7_shape= fc7.get_shape().as_list()

new_outputs=2
weights = tf.Variable(tf.truncated_normal([fc7_shape[3], num_outputs], stddev=0.05))
biases = tf.Variable(tf.constant(0.05, shape=[num_outputs]))
output = tf.matmul(fc7, weights) + biases
pred = tf.nn.softmax(output)

# Now, you run this with fine-tuning data in sess.run()
```

......
......
```python
saver = tf.train.import_meta_graph('vgg.meta')
```
Access the graph
```python
* graph = tf.get_default_graph()
```
* Prepare the feed_dict for feeding data for fine-tuning

* Access the appropriate output for fine-tuning
```python
fc7= graph.get_tensor_by_name('fc7:0')
```
* use this if you only want to change gradients of the last layer

```python
fc7 = tf.stop_gradient(fc7) # It's an identity function
fc7_shape= fc7.get_shape().as_list()

new_outputs=2
weights = tf.Variable(tf.truncated_normal([fc7_shape[3], num_outputs], stddev=0.05))
biases = tf.Variable(tf.constant(0.05, shape=[num_outputs]))
output = tf.matmul(fc7, weights) + biases
pred = tf.nn.softmax(output)

# Now, you run this with fine-tuning data in sess.run()
```
以上就是全部翻译，但是在实际处理过程中遇到了一些问题，实际上在深度学习的应用过程中我们的变量可能定义如下：
```python
with tf.name_scope('input'):
    x = tf.placeholder(tf.float32, [None, 784],name = 'x-input')
    y = tf.placeholder("float", [None, 10],name='y-input')
    x_image = tf.reshape(x,[-1,28,28,1],name='img')
    tf.summary.image('image',x_image,20)
```
通过with结构进行区分，在此情况下get_tensor_by_name函数获取变量的过程中需要添加上with的结构进行区分，否则无法获取到变量．整个tensorflow模型的保存和加载就介绍如下．
