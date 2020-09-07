---
title: tensorflow-start
date: 2017-05-02 09:20:13
tags: tensorflow学习
categories: 学习
mathjax: true
---
很想把深度学习好好学一学，曾经弄了一段时间的tensorflow不过双系统用起来确实让人不是那么的开心，所以慢慢也就放弃了，另外也是由于没有搞清楚tensorflow的具体过程，所以总是觉得迷迷糊糊的，而且tensorflow的可视化工具看起来很高端，可是用起来却不是那么舒服，一直迷迷糊糊这样不太好，还是决定好好学一学，况且今天看到了一个教程觉得不错，就暂且记一下吧所有引用均来自博客：http://www.open-open.com/lib/view/open1479797022780.html 如有侵权联系删除：  
><center><img src=http://static.open-open.com/lib/uploadImg/20161122/20161122144342_428.gif>

上面这一张图描述的是tensorflow的工作流程，暂时还不能具体分析，等全部弄明白之后我会回来详细分析这张流程图．根据博客描述tensorflow的工作流程为：  
>* 建立一个计算图
>* 初始化变量
>* 创建会话
>* 在会话中运行图
>* 关闭会话

为了详细说明一下流程编写了如下代码：  

    import tensorflow as tf

    a=tf.placeholder(tf.int16,name='a')
    b=tf.placeholder(tf.int16,name='b')
    addition = tf.add(a,b)
    init = tf.initialize_all_variables()

    tf.scalar_summary('a',a)
    tf.scalar_summary('b',b)
    tf.scalar_summary('add',addition)
    merged = tf.merge_all_summaries()
    with tf.Session() as sess:
        writer=tf.train.SummaryWriter('/tmp/tensorflow-start',sess.graph)
        #sess.run(init)
        result = sess.run(merged,feed_dict={a:2,b:3})
        writer.add_summary(result,0)
        print "Addition:%i"%sess.run(addition,feed_dict={a:2,b:3})

    sess.close()
ok,我们下面分析一下代码对应的工作流程，首先是引入tensorflow库，然后建立一个计算图，这个计算图很简单，就是一个加法运算而已，这个计算图中有三个变量，分别为a,b,addition,建立计算图后进行初始化，然后运行计算图，再运行计算图的过程中给变量初始值，最后关闭计算图。当然，为了尝试tensorflow的优越性，我将输出结果以图的形式输出，结果为：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/graph_run%3D.png">

从图上我们可以直观的看到加法运算的流程图，由两个变量相加得到变量add，通过此方法可以获取各个变量的值以及相关关系和相关关系图。此外再这里要提一提tensorflow的可视化工具，这么强大的工具居然没有一个比较好的教程，所有的教程要么是官网的例子，要么就是官网的代码，不过实际上用起来也比较简单，主要有三种变量设置，另外可以设置嵌套的变量，以后有机会要试试，另外还有一点就是得到的结果也必须run一次才能得到结果。
