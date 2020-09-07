---
title: 关于openSFM代码学习
date: 2020-03-22 21:09:55
tags: 图像处理
categories: 学习
---
一直在找和学习关于SFM的资料和代码，从刚开始的bundler到后来的ceres-sovler然后再到上层的应用openMVG+openMVS再到现在的openSFM，通过对这些的学习，对于SFM的基本知识我自认为已经是掌握得差不多了，最近在对openSFM的学习过程中同时对openSFM又有了一些改造，下面我详细的对自己的学习过程进行一个介绍，同时也对openSFM进行一个入门性质的介绍：  
首先我先介绍一下关于openSFM的[编译](https://opensfm.readthedocs.io/en/latest/)实际上在官网中介绍得比较详细了，下面我进行一个简单的介绍，openSFM是一个依赖于部分C++库的由python编写的SFM代码，所以它既具有较高的效率，同时也能够方便的进行接口的扩展和使用：
## 安装与编译
```github
git clone --recursive https://github.com/mapillary/OpenSfM
如果在clone的时候没有添加recursive参数，则后续可以通过如下处理

cd OpenSfM
git submodule update --init --recursive
```
openSFM依赖库包括：openCV，openGV，Ceres Solver以及Numpy，SciPy，Networkx, PyYAML, exifread其他的依赖库都没有什么好说的，主要说一下关于openGV的库，我选择的是ubuntu18.04 python3下安装，首先下载openGV的库，然后有一个很重要的步骤就是这是python安装录的设置，主要代码为：
```cmake

    mkdir -p /source && cd /source && \
    git clone https://github.com/paulinus/opengv.git && \
    cd /source/opengv && \
    git submodule update --init --recursive && \
    mkdir -p build && cd build && \
    cmake .. -DBUILD_TESTS=OFF \
             -DBUILD_PYTHON=ON \
             -DPYBIND11_PYTHON_VERSION=3.6 \
             -DPYTHON_INSTALL_DIR=/usr/local/lib/python3.6/dist-packages/ \
             && \
    make install && \
    cd / && rm -rf /source/opengv

```
以上参数：-DPYTHON_INSTALL_DIR主要目的在于设置python库的路径，这个根据自己的python安装库来进行设置，此外可能存在问题的部分在ceres solver的安装，一般给的地址都是google的地址，如果没有梯子无法clone下来，其实在github上也有副本，可以从github上clone下来然后编译，完成依赖库的安装后只需要
```python
pythonX setup.py build
```
其中X为自己安装python 的版本号。
## 简单使用
完成安装后在bin目录下就会出现opensfm和opensfm_run_all文件
```sh
#!/usr/bin/env bash

set -e

DIR=$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )
PYTHON=${2:-python3}

echo "Running using Python command: $PYTHON"

$PYTHON $DIR/opensfm extract_metadata $1
$PYTHON $DIR/opensfm detect_features $1
$PYTHON $DIR/opensfm match_features $1
$PYTHON $DIR/opensfm create_tracks $1
$PYTHON $DIR/opensfm reconstruct $1
$PYTHON $DIR/opensfm mesh $1
$PYTHON $DIR/opensfm undistort $1
$PYTHON $DIR/opensfm compute_depthmaps $1
```
从以上的文件中我们可以看到处理顺序，通过python bin/opensfm comman parameters来运行，首先解析元数据，然后检测特征点，特征点匹配，查找连接点，光束法平差计算，计算mesh，影像核线校正，然后计算深度图，实际上如果不做密集匹配后面三个步骤就不需要了，而而输入的参数实际上就是影像路径，影像文件夹中应该将影像放在images文件夹下，在解析过程中会自动查找images文件夹，找到文件夹下所有影像进行处理。
## 代码分析
python的代码相对简单，在这里我只针对核心代码进行分析，首先是
* config.py文件，这个文件设置了所有默认参数，在测试数据集中我们可以看到有一个config.yaml文件，实际上这个文件就是设置参数文件，实际上不是所有参数都需要设置，如果没有设置参数，则参数选取config.py中设置的默认参数进行处理，所以这个文件十分重要；
* dataset.py文件，这个文件是对数据集的定义文件，文件包含了所有文件的读取，保存构建影像列表等信息，因此如果需要对源码进行分析可以从输出文件往前分析；

其他的文件主要是功能性文件，具体代码实际上比较简单，在这里我就不再进行更加详细的介绍，如果有兴趣的话可以一起讨论关于openSFM的问题：wuwei_cug@163.com
## 输出文件分析
```json

reconstruction.json: [RECONSTRUCTION, ...]

RECONSTRUCTION: {
    "cameras": {
        CAMERA_ID: CAMERA,
        ...
    },
    "shots": {
        SHOT_ID: SHOT,
        ...
    },
    "points": {
        POINT_ID: POINT,
        ...
    }
}

CAMERA: {
    "projection_type": "perspective",  # Can be perspective, brown, fisheye or equirectangular
    "width": NUMBER,                   # Image width in pixels
    "height": NUMBER,                  # Image height in pixels

    # Depending on the projection type more parameters are stored.
    # These are the parameters of the perspective camera.
    "focal": NUMBER,                   # Estimated focal length
    "k1": NUMBER,                      # Estimated distortion coefficient
    "k2": NUMBER,                      # Estimated distortion coefficient
    "focal_prior": NUMBER,             # Initial focal length
    "k1_prior": NUMBER,                # Initial distortion coefficient
    "k2_prior": NUMBER                 # Initial distortion coefficient
}

SHOT: {
    "camera": CAMERA_ID,
    "rotation": [X, Y, Z],      # Estimated rotation as an angle-axis vector
    "translation": [X, Y, Z],   # Estimated translation
    "gps_position": [X, Y, Z],  # GPS coordinates in the reconstruction reference frame
    "gps_dop": METERS,          # GPS accuracy in meters
    "orientation": NUMBER,      # EXIF orientation tag (can be 1, 3, 6 or 8)
    "capture_time": SECONDS     # Capture time as a UNIX timestamp
}

POINT: {
    "coordinates": [X, Y, Z],      # Estimated position of the point
    "color": [R, G, B],            # Color of the point
    "reprojection_error": NUMBER   # Average reprojection error
}
```
以上就是[官网](https://opensfm.readthedocs.io/en/latest/dataset.html#reconstruction-file-format)关于renconstruction的解释，如果影像中包含经纬度信息还有reference_ll等中心经纬度信息，实际上shot就是影像，shot_id实际上就是影像名，**POINT实际上就是通过连接点计算得到的三维点信息，而POINT的id实际上就是trackID，据此可以时间track中影像坐标与POINT的三维点坐标进行匹配；** 