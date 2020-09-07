---
title: RPC几何校正预备工作
date: 2017-02-23 18:39:16
tags: 图像处理数学原理
categories: 图像处理
---
最近在有个朋友在做关于RPC校正的问题，实际上GDAL库有关于RPC校正的函数，在CSDN博客上也有[大神][1]进行了一些分析，不过这些分析都直接面向RFM参数进行校正的小伙伴，对于我这样刚把RPC原理搞清楚的人看起来有些难度且有一些东西并不是我所关心的，促使我进一步对RPC进行了解的一个契机是小伙伴给我发了两张ZY-3的影像数据，这些数据并不是原始数据而是裁剪了一部分的数据，因此在使用RPC参数进行校正时要加上偏移，实际上GDAL校正并没有给参数，所以想着自己写一个RPC几何校正，能够获取影像偏移，根据影像偏移进行校正。  
[1]: http://blog.csdn.net/liminlu0314/article/details/36206453  
下面首先来分析RPC几何校正的原理，一般来说对于影像校正，如果直接给定了姿态和位置参数的话就可以直接进行共线条件方程的严格几何校正，然而为了隐藏传感器的位置和姿态参数，且需要给用户提供一套几何校正参数，则RPC模型边应运而生，RPC模型建立的是影像坐标x,y到地理坐标X,Y,Z之间的变化关系：
<center><img src="http://latex.codecogs.com/gif.latex? r_n=\frac{P_1(X_n,Y_n,Z_n)}{P_2(X_n,Y_n,Z_n}(1)"/>
<center><img src="http://latex.codecogs.com/gif.latex? c_n=\frac{P_3(X_n,Y_n,Z_n)}{P_4(X_n,Y_n,Z_n}(2)"/>

上式中r,c和X，Y，Z为经过平移和缩放之后的标准化地理坐标和图像坐标其值为[-1,1]，坐标标准化的变换方法为：
<center><img src="http://latex.codecogs.com/gif.latex? X_n=\frac{X-X_0}{X_s}(3)"/>
<center><img src="http://latex.codecogs.com/gif.latex? Y_n=\frac{Y-Y_0}{Y_s}(4)"/>
<center><img src="http://latex.codecogs.com/gif.latex? Z_n=\frac{Z-Z_0}{Z_s}(5)"/>
<center><img src="http://latex.codecogs.com/gif.latex? c_n=\frac{c-c_0}{c_s}(6)"/>
<center><img src="http://latex.codecogs.com/gif.latex? r_n=\frac{r-r_0}{r_s}(7)"/>

其中<img src="http://latex.codecogs.com/gif.latex? X_0,Y_0,Z_0,c_0,r_0"/>为标准化的平移参数，<img src="http://latex.codecogs.com/gif.latex? X_s,Y_s,Z_s,c_s,r_s"/>为标准化的比例参数，RPC模型采用标准化坐标的目的是减少计算过程中由于数据数量级差别过大而引起的舍入误差，一般来说式(1),(2)中各个多项式最高次不大于3次，每一项各个坐标分量的幂总和也不大于3，且式(1),(2)中四个多项式的形式为：
<center><img src="http://latex.codecogs.com/gif.latex? P_1(X,Y,Z)=a_0+a_1X+a_2Y+a_3Z+a_4XY+a_5XZ+a_6YZ+a_7X^2+a_8Y^2+a_9Z^2+a_{10}XYZ+a_{11}X^2Y+a_{12}X^2Z+a_{13}Y^2X+a_{14}Y^2Z+a_{15}XZ^2+a_{16}YZ^2+a_{17}X^3+a_{18}Y^3+a_{19}Z^3(8)"/>

另外<img src="http://latex.codecogs.com/gif.latex? P_2,P_3,P_4"/>的形式与<img src="http://latex.codecogs.com/gif.latex? P_1"/>的形式类似只是参数a有所差别，一般来说通过二阶模型就能够改正投影畸变，地球曲率，大气折光以及镜头畸变的影响。以上推导参考论文[SPOT影像的RPC模型纠正][2]。  
[2]: http://xueshu.baidu.com/s?wd=paperuri:(ccad2328a715139938c1fb7cb4a9264d)&filter=sc_long_sign&sc_ks_para=q%3DSPOT%E5%BD%B1%E5%83%8F%E7%9A%84RPC%E6%A8%A1%E5%9E%8B%E7%BA%A0%E6%AD%A3&tn=SE_baiduxueshu_c1gjeupa&ie=utf-8&sc_us=15467269843722770795  
通过式（1）-（8）我们充分了解了RPC模型的基础上考虑影像RPC模型的校正，通过上式可得RPC模型给出的是地理坐标到影像坐标之间的变换关系，即给一个地理坐标可以计算影像坐标，但是在几何校正的过程中一般来说是给定影像坐标和RPC参数求解地面坐标，实际上我们看到，在校正过程中无法通过影像坐标求解地面坐标，地面坐标有三个未知数只有两个方程，但是在给定高程的基础上地面坐标是可以通过迭代的方法求解的。假定地面坐标X,Y为影像坐标的函数，则通过隐函数求导可以得到地面坐标与影像坐标的关系，然后在给定地面坐标初始值的情况下进行求导，得到地面坐标，其过程为：
<center><img src="http://latex.codecogs.com/gif.latex? r_n=r_0+\frac{\partial r}{\partial X}dX+\frac{\partial r}{\partial Y}dY(8)"/>
<center><img src="http://latex.codecogs.com/gif.latex? c_n=c_0+\frac{\partial c}{\partial X}dX+\frac{\partial c}{\partial Y}dY(9)"/>

通过上式进行迭代求解直到误差达到给定的误差限则停止。通过迭代的方法可以求出X和Y。  
以上是理论分析，实际上通过GDAL源码以及[博客][3]可知，GDAL中有RPC正解和反解的函数，这样就节省了大量时间，通过GDAL源码的alg文件夹，我们看到<font color='FF0000000'>gdal_rpc.cpp</font>这个文件，这里都是RPC校正的相关代码，代码较多，直接找接口，我们看到有三个与RPC校正相关的接口：<font color='FF0000000'>GDALCreateRPCTransformer、GDALRPCTransform、GDALDestroyRPCTransformer </font>这三个接口分别实现了穿件RPC转换模型RPC转换以及销毁模型，在RPC转换接口中实现了从影像坐标像经纬度转换的功能以及从经纬度向影像坐标转换的功能，因此校正过程就变得比较简单了。直接通过反算模型计算影像经纬度范围，通过影像分辨率和经纬度范围进行反算，最后将经纬度转换到WGS84投影坐标系下。
[3]: http://blog.csdn.net/liminlu0314/article/details/52302191
