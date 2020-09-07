---
title: python解析二进制文件
date: 2017-07-02 09:49:54
tags: 机器学习，图像处理
categories: 学习
---
昨天做了一个小工具将数据转换为了二进制的文件。深度学习工具用的是tensorflow，所以考虑如何在python下读取二进制文件，由于对Python并不是很熟，因此花了一番功夫。顺便记录一下以后有同样的处理也能够进行方便的处理。  
好了废话不多说，直接上代码：
```
import os
import struct
import random

datapath = '/home/wuwei/Picture/data.bin'
block_size = 4*50*50

def GetRandomROIs(int randRoiNumbers):
    input = open(datapath, 'rb')
    size = os.path.getsize(datapath)
    number = size/block_size

    randomMax = number - randRoiNumbers-1
    randomMin = 1
    if(randomMax<randomMin)
        return null

    int stPos = random.randint(randomMin, randomMax)*block_size;
    input.seek(stPos*block_size)
    binnarydata = input.read(randRoiNumbers*block_size)

    fmt = '<%di' %(randRoiNumbers*block_size/4)
    data = struct.unpack(fmt,binnarydata);
    input.close()
    return data
```
下面我们分析以上代码：  
首先导入必要的库文件，包括os库，struct，random库，首先使用open函数打开二进制文件，然后通过getsize函数获取数据的大小，为什么要获取数据大小，主要原因在于，每一个影像块的大小是固定的，因此可以通过数据大小和块大小得到影像块个数，而函数的输入参数是得到多少个影像块，因此可以随机选取读取的开始位置，然后通过read函数读取相应的大小。read函数比较简单，但是有一点需要注意，那就是读取的时候需要以字节为单位如果读取一个int则read函数的参数为4,读取的过程也比较简单，读取数据后依然是二进制的数据，需要按照要求进行转换，转换的代码为：
```
fmt = '<%di' %(randRoiNumbers*block_size/4)
data = struct.unpack(fmt,binnarydata);
```
解析主要是通过unpack来实现的，fmt为解析的格式，格式的描述可以参看网站：https://docs.python.org/2/library/struct.html
实际上如果是对多个数据进行解析也可以直接加数字再前面，由于这个解析的大小不是固定的，因此通过fmt构造解析字符串，通过解析字符串解析每一个int类型的数据将其保存在data中，因此得到的data中梅25×25大小的数据为一个影像块，由此可以解析每一个影像块作为深度学习的输入数据。
