---
title: fisher 判别式详细推导
date: 2017-04-20 22:24:24
tags: 数学
categories: 数学
mathjax: true
---
昨天晚上跟师妹推导fisher判别式的时候，推导到一半的时候突然觉得有点不对劲，然后看网上各种博客也总觉得有些问题，有一个步骤总是觉得不太对，只好自己仔细思考一下了，最后发现自己的推导结果并没有问题，实际上有一个步骤出了一点点小问题所以没有办法得到最后的结果，正好借着这个机会也总结一下fisher判别的过程，实际上，fisher判别的过程还是比较简单的，总的来说就是将数据投影到一条线上，然后使得投影后的数据具有类内方差极小，类间差异极大的特点，如图：
<center><img src = https://lh3.googleusercontent.com/Xc4akeGc9efjU15Ixb8ssQlGTHm7ywvUpqNRMsRJCl09Su1JOFAW9tNp7PuphqaJatKe3uQC=s0 ></center>
其中可以明显的看出有两类，在二维空间中，实际上这两类是通过模拟得到的，简单的来说，fisher判别式，也就是我们所说的LDA就是通过一条投影线将ｎ维空间中的数据投影到一维空间，则投影公式为:
<center><img src="http://latex.codecogs.com/gif.latex? y=w^tx"/> (1)

其中ｙ为投影后的坐标，ｘ为投影前的坐标,w为投影向量，则实际上就是求ｗ使得投影后数据ｙ具有最大类间差异和最小类内差异，则类间差异通过两类的均值差异表示，而类内的聚合度则通过类内的方差表示，则实际上就是求以下表达式的极大值：
<center><img src="http://latex.codecogs.com/gif.latex? J(w)=\frac{m_y}{S_y}"/>    (2)

式（２）中$m_y$为两类的均值差，$S_y$为两类的方差之和，则有如下式子：
<center><img src="http://latex.codecogs.com/gif.latex? m_y=m^{*}_1-m^{*}_2" /> (3)
<center><img src="http://latex.codecogs.com/gif.latex? S_y=S^{*}_1-S^{*}_2" /> (4)

其中式（３）（４）为投影后的两类的均值差和标准差之和，而实际上，投影后的均值和标准差可以通过式（１）得到，因此将式（１）（３）（４）代入式（２），由于取差值的大小，则通过对式（３）取平方项，式（２）可变形为:
<center><img src="http://latex.codecogs.com/gif.latex? J(w)=\frac{w^{t}m_xw}{w^{t}S_xw}"/> (5)

而实际上就是求式(5)的极大值，从上式一直到式（５）都是投影后类内方差最小，类间距最大的自然推导结果，而下面的推导才是重点，实际上对于式（５）我们有其他的表现形式，式（５）中必须假设$S_x$为非奇异矩阵，
则式（５）可写做：$J(w)=w^{t}m_xw-\lambda{w^{t}}S_xw$,其中$\lambda$为调节均类间差异度和类内聚合度的变量，则对式上式求导可得：
<center><img src="http://latex.codecogs.com/gif.latex? J(w)=2w^{t}m_x-2\lambda{w^t}S_x"/>

则我们可以将ｗ看作矩阵 $m_{x}S^{-1}_x$ 的特征向量，而 $\lambda$ 为特征值，在实际求解的过程中之需要知道投影向量的方向，而不需要知道其大小，则实际上有 $w=(m_1-m_2)S^{-1}_x$
