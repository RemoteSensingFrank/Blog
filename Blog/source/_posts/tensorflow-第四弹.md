---
title: tensorflow-第四弹
date: 2017-07-24 22:25:01
tags: tensorflow学习
categories: 学习
mathjax: true
---
前面三个部分都是关于深度学习，tensorflow框架的东西，再接着深入就要涉及到了深度学习更加深层次的内容了，涉及到深度卷积和深度置信网络了，这一部分涉及的原理比较多，因此相对来说会比较复杂，而为了进行深度学习我们必须在另一个部分进行比较深入的研究，那就是数据，深度学习必然伴随这大数据，可以说没有大数据就没有深度学习。我们接下来学习跟数据有关的一些内容，主要就是针对数据获取和数据预处理两个部分。  
我们知道网络上的数据很多，也很复杂，有各种标签的数据，这些数据有的对我们有用，而有些数据对我们来说就没有什么意义了，如何快速获取大量对我们来说有意义的数据是我们亟待解决的问题，所以这一部分看似和tensorflow没有什么关系，实际上它是机器学习的基础，所以我们再后面一部分会着重的介绍数据获取和数据处理，直到这一部分学习完才会继续真正的深度学习方法的学习和介绍，好了废话不多说。  
我们这次主要讲爬虫，什么是爬虫或者说robot，简单的来说就是能够从网上获取数据的工具，而这样的工具都是通过代码实现的，我们把这样的代码或者工具就叫做爬虫。好了介绍完爬虫我们基本上就知道爬虫是干什么的了，没错爬虫就是替代我们从网上获取数据的，那么它的工作原理是怎么样的呢，我们会通过两个网站的分析进行介绍：  
 * 京东数据爬虫：  
 说到第一个爬虫必须详细介绍一下整个分析过程，在进行数据爬取过程中有一个很重要的过程那就是网站的分析，对我们需要爬取的数据的分析，我们简单的看一下京东的网站特点：
 ``` html
 <li class="gl-item" data-sku="10979250498" data-spu="10171150160" data-pid="10979250498">
	<div class="gl-i-wrap">
		<div class="p-img">
			<a target="_blank" href="//item.jd.com/10979250498.html" onclick="searchlog(1,10979250498,22,2,'','flagsClk=1614807180')">
				<img width="220" height="220" class="err-product" data-img="1" data-lazy-img="//img13.360buyimg.com/n7/jfs/t6349/155/1512158350/203785/22921fdf/595224a5N9d0a2565.jpg" />
			</a>
			<div data-catid="655" data-jq="1" data-dq="1" data-venid="608833" data-template="868771_n" data-promise="600_0"></div>
		</div>
		<div class="p-scroll">
			<span class="ps-prev">&lt;</span>
			<span class="ps-next">&gt;</span>
			<div class="ps-wrap">
				<ul class="ps-main">
					<li class="ps-item"><a href="javascript:;" class="curr" title="玫瑰金"><img data-sku="10979250498" data-img="1" data-lazy-img="//img13.360buyimg.com/n9/jfs/t6349/155/1512158350/203785/22921fdf/595224a5N9d0a2565.jpg" class="err-product" width="25" height="25" /></a></li>
										<li class="ps-item"><a href="javascript:;" title="香槟金"><img data-sku="10979250499" data-img="1" width="25" height="25" data-lazy-img="//img14.360buyimg.com/n9/jfs/t6436/43/1478836466/244215/e2aff1c7/595224b2N8b62ebb3.jpg" class="err-product" /></a></li>
									</ul>
			</div>
		</div>
		<div class="p-price">
<strong class="J_10979250498" data-price="629.00"><em>¥</em><i>629.00</i></strong>		</div>
		<div class="p-name p-name-type-2">
			<a target="_blank" title="小米 红米4A 全网通 4G手机 玫瑰金 移动4G/联通4G/电信4G( 2G+16G)" href="//item.jd.com/10979250498.html" onclick="searchlog(1,10979250498,22,1,'','flagsClk=1614807180')">
				<em>小米 红米4A 全网通 4G<font class="skcolor_ljg">手机</font> 玫瑰金 移动4G/联通4G/电信4G( 2G+16G)</em>
				<i class="promo-words" id="J_AD_10979250498"></i>
			</a>
		</div>
		<div class="p-commit">
			<strong>已有<a id="J_comment_10979250498" target="_blank" href="//item.jd.com/10979250498.html#comment" onclick="searchlog(1,10979250498,22,3,'','flagsClk=1614807180')">7200+</a>条评价</strong>
		</div>
		<div class="p-focus"><a class="J_focus" data-sku="10979250498" href="javascript:;" title="点击关注" onclick="searchlog(1,10979250498,22,5,'','flagsClk=1614807180')"><i></i>关注</a></div>
		<div class="p-shop" data-selfware="0" data-score="5" data-reputation="95" data-shopid="602745"></div>
		<div class="p-icons" id="J_pro_10979250498">
			<i class="goods-icons J-picon-tips J-picon-fix" data-tips="京东发货，商家提供售后服务">京东配送</i>
		</div>
		<div class="p-addtocart hide"><a data-stock="10979250498" data-stock-val="1" data-presale="0"></a></div>
	</div>
</li>
<li class="gl-item" data-sku="10424732616" data-spu="10092705106" data-pid="10424732616">
	<div class="gl-i-wrap">
		<div class="p-img">
			<a target="_blank" href="//item.jd.com/10424732616.html" onclick="searchlog(1,10424732616,23,2,'','flagsClk=1614807688')">
				<img width="220" height="220" class="err-product" data-img="1" data-lazy-img="//img11.360buyimg.com/n7/jfs/t6067/294/5480602146/435008/896f90bc/596c13c1Ne4dd122a.jpg" />
			</a>
			<div data-catid="655" data-jq="1" data-dq="1" data-venid="190328" data-promise="330_0"></div>
		</div>
		<div class="p-scroll">
			<span class="ps-prev">&lt;</span>
			<span class="ps-next">&gt;</span>
			<div class="ps-wrap">
				<ul class="ps-main">
					<li class="ps-item"><a href="javascript:;" class="curr" title="橙色"><img data-sku="10424732616" data-img="1" data-lazy-img="//img11.360buyimg.com/n9/jfs/t6067/294/5480602146/435008/896f90bc/596c13c1Ne4dd122a.jpg" class="err-product" width="25" height="25" /></a></li>
										<li class="ps-item"><a href="javascript:;" title="黑色"><img data-sku="10424732618" data-img="1" width="25" height="25" data-lazy-img="//img13.360buyimg.com/n9/jfs/t5650/199/6735451344/399114/a807ca0f/596c1404N200d8049.jpg" class="err-product" /></a></li>
										<li class="ps-item"><a href="javascript:;" title="军绿色"><img data-sku="10424732617" data-img="1" width="25" height="25" data-lazy-img="//img12.360buyimg.com/n9/jfs/t6118/118/5549490934/419480/e566b797/596c13deN659fbf22.jpg" class="err-product" /></a></li>
									</ul>
			</div>
		</div>
		<div class="p-price">
<strong class="J_10424732616" data-price="268.00"><em>¥</em><i>268.00</i></strong>		</div>
		<div class="p-name p-name-type-2">
			<a target="_blank" title="中兴健康（ZTE Health）L628  移动/联通2G 三防直板老人手机 橙色" href="//item.jd.com/10424732616.html" onclick="searchlog(1,10424732616,23,1,'','flagsClk=1614807688')">
				<em>中兴健康（ZTE Health）L628  移动/联通2G 三防直板老人<font class="skcolor_ljg">手机</font> 橙色</em>
				<i class="promo-words" id="J_AD_10424732616"></i>
			</a>
		</div>
		<div class="p-commit">
			<strong>已有<a id="J_comment_10424732616" target="_blank" href="//item.jd.com/10424732616.html#comment" onclick="searchlog(1,10424732616,23,3,'','flagsClk=1614807688')">1.5万+</a>条评价</strong>
		</div>
		<div class="p-focus"><a class="J_focus" data-sku="10424732616" href="javascript:;" title="点击关注" onclick="searchlog(1,10424732616,23,5,'','flagsClk=1614807688')"><i></i>关注</a></div>
		<div class="p-shop" data-selfware="0" data-score="5" data-reputation="96" data-shopid="184371"></div>
		<div class="p-icons" id="J_pro_10424732616">
			<i class="goods-icons J-picon-tips J-picon-fix" data-tips="京东发货，商家提供售后服务">京东配送</i>
		</div>
		<div class="p-addtocart hide"><a data-stock="10424732616" data-stock-val="1" data-presale="0"></a></div>
	</div>
</li>
 ```
截取了一部分进行分析，我们看到其实所有手机列表都在标签\<li>\</li>之间，然后我们进行两步匹配，所谓的两步匹配就是首先通过正则表达式<sub>[1]</sub>匹配所有\<li>\</li>之间的数据，然后在进行进一步分析我们看到，所有手机的图片都在其中的<img>标签内，根据这样我们可以找到所有图片的链接，最后根据图片链接获取图片。好了一个网页我们就获取到了，那么如何获取下一页呢，这个就要分析网页的url了，我们分析京东手机网站的url：https://search.jd.com/Search?keyword=%E6%89%8B%E6%9C%BA&enc=utf-8&wq=%E6%89%8B%E6%9C%BA    
分析上面那个网站，可以很清楚的看出，搜索的过程，更进一步，我们看到下一页：
https://search.jd.com/Search?keyword=%E6%89%8B%E6%9C%BA&enc=utf-8&qrst=1&rt=1&stop=1&vt=2&wq=%E6%89%8B%E6%9C%BA&cid2=653&cid3=655&page=3&s=56  
好了，我们比较一下有那些区别，我们看到多了一些参数，其实这些参数大部分都没有什么用，除了那个page，没错，我们修改page后面就是我们要得到的下一页，那么问题就来了，为什么明明是第二页那个page的值是3,这就是京东的一个trick了，网站下一页是+2而不是+1,这个我们多看几个网站就能找到规律了，知道这个后我们就可以去爬取网站了，每次爬取page的一个网站，从网站中解析所有img的url，然后下载下来，保存到本地，好了，一个简单的明文网站的爬取就到这里了，下面我们介绍难一点的网站，比如百度图片  
我们从百度上搜一个图片，然后查看网页源码我们可以看到，与京东的明显的网页不一样的是，百度的网页都是脚本代码，实际上我们还是可以得到一些有用信息的，这里就不详细分析了，实际上百度搜索返回的是json格式的数据，所以我们目标是解析json格式数据得到url，不过只要有就不怕解析不到，但是这里有一个小技巧必须要提一下，我们用图片搜索的时候并不是像京东那样下一页的变化，所以直接从地址也看不到什么，因为其是相应式的，所以我们进入调试模式看它request提交得到的东西，我们看到实际上它提交的是：http://image.baidu.com/search/avatarjson?tn=resultjsonavatarnew&ie=utf-8&word=%E8%BE%B9%E7%89%A7&rn=60&pn=60
这样一个网站，同样有很多参数，但是同样参数对我们来说最有用的rn是pn后面的参数，实际上它确定了一次向后跳60页以及当前的页数，所以我们同样可以以这样的形式获取下一页的搜索结果，另外我们看搜索结果：
```json
{"lifeTrackHead":"", "pageType":"0", "setNumber":0, "IsAvatarQuery":"0", "avatarCategory":"", "tagZero":"", "tagOne":"", "tagTwo":"", "tagFatherSelected":"", "tagSelected":"", "tags":[], "simgs":[],"tag":"", "AVnum":"0", "imgtotal":"1985", "displayNum":"", "gsm": "78", "listNum":"1985", "headPic":{ "status":"", "query":"", "desc":"", "sign":"", "obj_url":"", "from_url":"", "url":"", "summary":"", "pageNum":"-1", "tagTwo":"", "thumbURL":"", "width":"", "height":""}, "imgs":[{ "thumbURL":"http://img1.imgtn.bdimg.com/it/u=3598937744,2157458349&fm=26&gp=0.jpg", "middleURL":"http://img1.imgtn.bdimg.com/it/u=3598937744,2157458349&fm=26&gp=0.jpg", "largeTnImageUrl":"", "hasLarge" : true, "hoverURL":"http://img1.imgtn.bdimg.com/it/u=3598937744,2157458349&fm=26&gp=0.jpg", "pageNum":60, "objURL":"http://c.hiphotos.baidu.com/zhidao/pic/item/f636afc379310a55b1d71ec4b44543a98226100a.jpg", "fromURL":"http://zhidao.baidu.com/question/1494493144181586139.html", "fromURLHost":"zhidao.baidu.com", "currentIndex":"", "width":2448, "height":3264, "type":"jpg", "filesize":"", "bdSrcType":"0", "di":"194609389151", "is":"0,0", "bdSetImgNum":0, "bdImgnewsDate":"1970-01-01 08:00", "fromPageTitle":"鎴戣寰楀簲璇ユ槸杈圭墽鍚�  杩欐槸鐜板湪鐨勬牱瀛�", "bdSourceName":"", "bdFromPageTitlePrefix":"", "isAspDianjing":false, "token":"", "source_type":"", "tagTwo":"", "cs":"3598937744,2157458349", "os":"4201772925,2284092193", "simid":"3198441173,82549856", "pi":"0", "adType":"0", "setDowloadURL":"", "setTittle":"", "DecorateCompanyName":"", "DecorateCompanyLocation":"", "DecorateWantuUrl":"", "DecorateCompanyId":"", "DecorateCompanyGrade":"", "personalized":"0" },{ "thumbURL":"http://img4.imgtn.bdimg.com/it/u=1150363191,333734041&fm=26&gp=0.jpg", "middleURL":"http://img4.imgtn.bdimg.com/it/u=1150363191,333734041&fm=26&gp=0.jpg", "largeTnImageUrl":"", "hasLarge" : true, "hoverURL":"http://img4.imgtn.bdimg.com/it/u=1150363191,333734041&fm=26&gp=0.jpg", "pageNum":61, "objURL":"http://h.hiphotos.baidu.com/zhidao/pic/item/728da9773912b31bb7f204fc8018367adbb4e14d.jpg", "fromURL":"http://zhidao.baidu.com/question/329112378851785325.html", "fromURLHost":"zhidao.baidu.com", "currentIndex":"", "width":1024, "height":768, "type":"jpg", "filesize":"", "bdSrcType":"0", "di":"148487110091", "is":"0,0", "bdSetImgNum":0, "bdImgnewsDate":"1970-01-01 08:00", "fromPageTitle":"璇风湅鐪嬫垜鐨�杈圭墽鏄笉鏄覆鐨�", "bdSourceName":"", "bdFromPageTitlePrefix":"", "isAspDianjing":false, "token":"", "source_type":"", "tagTwo":"", "cs":"1150363191,333734041", "os":"3716457608,3614219987", "simid":"3348089744,278242429", "pi":"0", "adType":"0", "setDowloadURL":"", "setTittle":"", "DecorateCompanyName":"", "DecorateCompanyLocation":"", "DecorateWantuUrl":"", "DecorateCompanyId":"", "DecorateCompanyGrade":"", "personalized":"0" },{ "thumbURL":"http://img4.imgtn.bdimg.com/it/u=3967093973,1537353609&fm=26&gp=0.jpg", "middleURL":"http://img4.imgtn.bdimg.com/it/u=3967093973,1537353609&fm=26&gp=0.jpg", "largeTnImageUrl":"", "hasLarge" : true, "hoverURL":"http://img4.imgtn.bdimg.com/it/u=3967093973,1537353609&fm=26&gp=0.jpg", "pageNum":62, "objURL":"http://h.hiphotos.baidu.com/zhidao/pic/item/d52a2834349b033bbc58afc11dce36d3d439bddd.jpg", "fromURL":"http://zhidao.baidu.com/question/1242156031728342739.html", "fromURLHost":"zhidao.baidu.com", "currentIndex":"", "width":780, "height":1040, "type":"jpg", "filesize":"", "bdSrcType":"0", "di":"89401290661", "is":"0,0", "bdSetImgNum":0, "bdImgnewsDate":"1970-01-01 08:00", "fromPageTitle":"鎴戝杩欎釜姣嶄覆涓插拰杈圭墽涓查厤绉嶇殑璇濅細鐢熷嚭杈圭墽鍚�", "bdSourceName":"", "bdFromPageTitlePrefix":"", "isAspDianjing":false, "token":"", "source_type":"", "tagTwo":"", "cs":"3967093973,1537353609", "os":"4070950924,522173132", "simid":"3378736389,427982830", "pi":"0", "adType":"0", "setDowloadURL":"", "setTittle":"", "DecorateCompanyName":"", "DecorateCompanyLocation":"", "DecorateWantuUrl":"", "DecorateCompanyId":"", "DecorateCompanyGrade":"", "personalized":"0" },{ "thumbURL":"http://img4.imgtn.bdimg.com/it/u=2088893379,3929280629&fm=26&gp=0.jpg", "middleURL":"http://img4.imgtn.bdimg.com/it/u=2088893379,3929280629&fm=26&gp=0.jpg", "largeTnImageUrl":"", "hasLarge" : true, "hoverURL":"http://img4.imgtn.bdimg.com/it/u=2088893379,3929280629&fm=26&gp=0.jpg", "pageNum":63, "objURL":"http://g.hiphotos.baidu.com/zhidao/pic/item/9358d109b3de9c828f5e8e3e6b81800a19d84343.jpg", "fromURL":"http://zhidao.baidu.com/question/1671228469381951107.html", "fromURLHost":"zhidao.baidu.com", "currentIndex":"", "width":780, "height":1040, "type":"jpg", "filesize":"", "bdSrcType":"0", "di":"185194576271", "is":"0,0", "bdSetImgNum":0, "bdImgnewsDate":"1970-01-01 08:00", "fromPageTitle":"鎳�杈圭墽鐨勬潵 璋佸彲浠ュ憡璇夋垜 鎴戝鐙楃嫍鍒板簳鏄笉鏄拰鑻忕墽涓茬殑 寰堝枩娆� 涓�", "bdSourceName":"", "bdFromPageTitlePrefix":"", "isAspDianjing":false, "token":"", "source_type":"", "tagTwo":"", "cs":"2088893379,3929280629", "os":"4196081066,1970071337", "simid":"4194815420,749294635", "pi":"0", "adType":"0", "setDowloadURL":"", "setTittle":"", "DecorateCompanyName":"", "DecorateCompanyLocation":"", "DecorateWantuUrl":"", "DecorateCompanyId":"", "DecorateCompanyGrade":"", "personalized":"0" },{ "thumbURL":"http://img1.imgtn.bdimg.com/it/u=4106747,2199010286&fm=26&gp=0.jpg", "middleURL":"http://img1.imgtn.bdimg.com/it/u=4106747,2199010286&fm=26&gp=0.jpg", "largeTnImageUrl":"", "hasLarge" :
```
从结果我们可以看到都是一些json的数据，但是我们依然可以从其中挖掘信息，我们看到图片都存在objurl的标签中，所以我们通过正则表达式获取其中图片链接，并下载图片，通过以上的分析我们可以了解到，通过爬虫获取数据最重要的是知道目标，然后是解析和存储，这就是一个简单爬虫工具的编写，下一步将介绍爬虫框架，写出较完整的爬虫工具。
我们看代码：实际上爬取JD的代码被注释了：
```python
import re
import urllib
import urllib2
import time

import socket
socket.setdefaulttimeout(20.0)

poxy_addr=[
"61.185.219.126:3128"
"218.247.161.37:80"
"218.75.100.114:8080"
"222.68.207.11:80"
"221.204.246.116:3128"]
def craw(url,page):
    print(url)
    headers=("User-Agent","Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:53.0) Gecko/20100101 Firefox/53.0")
    poxy_id = 0
    #poxy = urllib2.ProxyHandler({'http':poxy_addr[poxy_id]})
    opener = urllib2.build_opener()
    opener.addheaders=[headers]
    #urllib2.install_opener(opener)

    try:
        html1 = opener.open(url).read()
    except Exception as e:
            return
    """
    html1 = ""
    for i in range(5):
        try:
            html1 = opener.open(url).read()
        except urllib2.URLError as e:
            poxy_id=(poxy_id+1)%5
            print(poxy_id)
    """
    #print (html1)
    #html1 = str(html1)
    partten1 = '"objURL":"(.+?)",'
    fliter1 = re.compile(partten1).findall(html1)
    x=1
    for imageurl in fliter1:
        imagename = 'Baidu_Dog/'+str(page)+str(x)+'.jpg'
        #imageurl = 'http://'+imageurl
        print("image:"+imageurl)
        print("save:"+imagename)
        try:
            urllib.urlretrieve(imageurl,imagename)
        except urllib2.URLError as e:
            if(hasattr(e,"code")):
                x+=1
            if(hasattr(e,"reason")):
                x+=1
        except Exception as e:
            continue
        x+=1
    """
    for fliteri in fliter1:
        partten2 = '<img width="220" height="220" class="err-product" data-img="1" src="//(.+?)" />'
        imglist = re.compile(partten2).findall(fliteri)

        x = 1
        for imageurl in imglist:
            imagename = 'JD_Telphone/'+str(page)+str(y)+str(x)+'.jpg'
            imageurl = 'http://'+imageurl
            print("image:"+imageurl)
            print("save:"+imagename)
            try:
                urllib.urlretrieve(imageurl,filename=imagename)
            except urllib2.URLError as e:
                if(hasattr(e,"code")):
                    x+=1
                if(hasattr(e,"reason")):
                    x+=1
                x+=1
        y+=1
    """
for i in range (60,3000,60):
    url = 'http://image.baidu.com/search/avatarjson?tn=resultjsonavatarnew&ie=utf-8&word=%E8%BE%B9%E7%89%A7&rn=60&pn='+str(i)
    craw(url,i)
```
代码比较简单也没有什么可以分析的，有一点要提醒一下，那就是python2.x和python3.x再urllib库的使用上是有差异的，具体的差异大家可以网上找找，避免由于版本的差异导致出错。
