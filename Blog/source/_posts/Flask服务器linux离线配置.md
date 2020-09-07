---
title: Flask服务器linux离线配置
date: 2018-06-20 07:41:11
tags: 学习
---
Flask为Python的一个轻量级服务器框架，用来做Restful风格的后台程序是十分合适的，再结合Nginx做负载均衡，则整个后台服务器能够承受较大规模的并发和用户访问，在项目中本来是在windows下开发的，不过实际部署环境是linux，弄得有些措手不及，不过好在整个Python框架都是支持跨平台的，所以不管是Windows还是Linux也都能够适用，只是整个环境配置会比较麻烦。  
涉及到的几个主要的组件为：
* 1.Flask以及相关组件
* 2.Tornado以及相关组件
* 3.SQLAlchemy以及相关组件

以上三个组件是比较重要的组件，由于每个组件又存在一些依赖关系，所在安装过程中显得比较麻烦，下面我依次介绍每一个组件的安装依赖，以便于在下次部署的过程中能够方便的进行部署：
一般来说离线安装组件都是通过pip install 来安装whl文件，因此首先需要安装setuptools以及pip组件。
### Flask组件的安装
实际上如果是在线环境，则使用yum或者apt-get安装都是极其方便的，在线环境下会自动下载各个依赖，但是在离线环境下就必须先下载安装好各个依赖然后才能够进行Flask的安装，Flask的依赖包括：  
Werkzeug>=0.7；
Jinja2>=2.4, which requires； 
MarkupSafe；
Babel>=0.8, which requires； 
pytz；
itsdangerous>=0.21；
实际上安装的过程比较简单，首先下载好离线包，如果没有whl文件就下载源码直接通过编译安装，如果又whl文件则通过pip根据whl文件进行安装，完成依赖库的安装后，就可以安装Flask了，安装的方法同样是下载whl文件然后通过pip install进行安装；
### Tornado组件的安装
Tornado的安装比较简单，首先下载源码，实际上tornado并没有找到whl文件所以只能够通过源码安装，安装的方式相对来说也比较简单，首先通过tar -zxvf命令解压文件，然后进入文件夹，这是应该又一个setup.py文件，直接通过python setup.py instll 安装tornado，安装过程中没有错误则说明安装完成；
### SQLAlchemy的安装
要通过python连接MySQL数据库，则需要一些组件，而且整个过程相对会复杂一些，首先需要安装python-devel，在安装的过程中一定要注意python的版本，下载的时候也最好下载对应操作系统，对应python版本的库，以免引起冲突；python-devel是一个rpm库，直接通过rpm -ivh命令进行安装，若在安装过程中没有提示错误，则说明安装成功，安装python运行库后需要安装mysqldb组件，首先下载python-MySQLdb压缩文件，进入压缩文件目录后可以看到有setup.py文件，直接通过python install进行安装就可以了。

完成以上所有组件的安装后直接运行服务，如果还缺某一个组件，则会提示import的头文件不存在，此时只需要将文件下载下来安装就可以了，整个配置过程是比较方便的，另外要注意的就是python连接数据库的组件，需要注意操作系统和python的版本，安装正确的版本才能够正确是使用