---
title: ArcGIS10.5.1安装与配置
date: 2017-11-12 12:31:29
tags: ArcGIS环境配置
categories: 学习
---
新版本的ArcGIS组件特别多，最新的版本就我了解已经到了10.5.1的版本了，新的版本带来了很强大的功能，同时也存在着使用文档和说明不足的问题，特别是对于广大类似我这样在学校一直只用Desktop10.1的小白用户，所以在这里先介绍一下ArcGIS 服务器和桌面端的版本情况，然后针对最新版的软件介绍一下其安装和部署的方案以及发布数据的方法。  
ok，闲话不多说直接进入ArcGIS版本的情况说明：  
目前来说ArcGIS两大系列的产品分别为ArcGIS Enterprise（以前的ArcGIS Server）以及ArcGIS Desktop其中ArcGIS Enterprise包括：ArcGIS Server，Portal for ArcGIS，Data Store，Web Adaptor;而ArcGIS Server 又包括：GIS Server，GeoAnalytics Server、GeoEvent Server、Image Server等，除了基础GIS Server为Enterprise基础授权以外其他的服务都是扩展服务；而实际上部署一个完整的地理信息系统的服务器需要GIS Server，Portal for ArcGIS，Data Store，Web Adaptor这些基础组件也就够了，对于其他的高级应用则需要其他的扩展服务，不过实际上目前的几个组件部署对于小白来说也算得上是一件比较麻烦的事情了，耗费了一天的时间之后终于完成了ArcGIS服务器的部署，由于部署过程个人感觉比较复杂，为了避免自己遗忘部署的过程，在这里将部署的过程详细记录一下希望下次能够给自己一点提示：  
首先上一张再服务器上完成所有组件安装的截图：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/%E5%AE%89%E8%A3%85%E6%96%87%E4%BB%B6%E6%95%B4%E4%BD%93%E5%9B%BE.jpg" align="center">  
首先为软件的安装：软件的安装包括1)安装ArcGIS Server；2)安装Portal for ArcGIS；3)安装License Manager；安装完成这三个软件之后剩下的功能就是激活了，激活的方法如果是正版就很简单，直接根据序列号进行激活，如果可以连接外网则直接选择在线激活，如果无法连接外网则需要导出授权文件，然后对将授权文件拷贝到可以上网的机器上，上传到ArcGIS验证服务器获取授权文件，获取授权文件进行授权。完成安装和授权之后就算完成了服务器配置的第一步，下面就要进入正式的配置环节了。  
Web Adaptor软件的配置：设置GIS作为Portal的托管服务器则必须为Portal配置web adaptor，对于ArcGIS来说Web Adaptor使用443和80两个端口，具体端口的选用可以根据自己的选择，安装web Adaptor后需要通过IIS构建绑定，实际上这里有一个开通IIS的过程，在安装的过程中会自动检测需要的IIS组件，在控制面板中将IIS相应的组件打开就好了,安装完成web adaptor之后就会有配置的提示如图：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/portal%E7%BC%96%E8%BE%91.jpg">  
将portal的url输入然后点击配置就可以完成web adaptor的配置，配置完成后如上图会出现提示网址，点击网址如果能够打开Pｏrtal就说明portal配置成功，在配置成功Portal之后需要将Portal与Server进行连接，连接的过程如图：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/portal%E8%81%94%E5%90%88%E6%9C%8D%E5%8A%A1%E5%99%A8.png">  
实际上并没有什么要特别注意的，有两点需要提一下啊，首先portal在配置的过程中需要通过web adaptor配置的域名登录才行，另外就是与Server连接的过程中需要首先安装Data store，这个组件也很好安装，按照给的提示一步步的安装应该就能够将组件安装成功，另外如果购买的是正版的ArcGIS Enterprise则Data Store与web adaptor就包含在其中了，如果Server与Portal安装在同一台服务器上则Server的地址为需要带端口的地址，同时需要使用https协议的地址，输入之后就可以完成联合，完成联合之后在选择host服务器的位置选择我们刚刚联机的的Server的服务器就好了，完成之后就算完成了Portal与Server的联合，之后在Portal上共享的数据就可以通过Server进行发布了．
