---
title: tensorflow-第五弹
date: 2017-07-27 22:19:54
tags: tensorflow学习
categories: 学习
mathjax: true
---
今天加了个班，回来就差不多只能去写写博客总结一下了，以前老师觉得写博客是浪费时间。不过现在想想好像也还好啦，一直在学习的过程中也需要时常进行一下总结。所谓学而不思则罔嘛，昨天调通了多线程的爬虫代码，今天来总结一下，好了废话不多说我们直接进入主题。  
多线程对我们来说已经不用解释了，我们重点要讲的是如何通过python实现多线程以及多线程爬虫的设计，首先总结一下通过python进行多线程的代码的编写，我们首先来看两个demo：（算了，demo删掉了）我们就直接分析python多线程的写法吧，首先需要import threading这个模块，我们需要定义一个类，此类继承自threading类，然后实现run函数，将线程操作放在run函数中，这一点不知道是不是必要的，不过demo都在run函数中，所以也不做它想以后干脆都写在run函数中算了，这样一个线程类就写好了，在python中启动线程也比较简单，直接start就能够启动线程了，python的线程是轻量级的所以创建和启动都比较容易，至于有什么缺点，这个暂时不去研究了。知道了线程的创建方法之后我们下一步去研究一下如何将爬虫代码修改为多线程，我们采用上次的百度图片的爬虫代码进行研究。  
我们分析爬虫代码的耗时过程可以发现主要有两个耗时的过程，第一个为解析，从html，百度的json数据中解析出需要的url是一个比较耗时的过程，另外一个是图片的获取和保存是一个耗时的过程，由于有这两个耗时过程，我们可以对这两个过程进行拆分，将其拆分为两个线程并行处理，简单的思路就这样，我们下面对代码进行分析：
```python
import threading
import Queue as queue
import re
import urllib2
import urllib
import time

urlqueue = queue.Queue()
headers=("User-Agent","Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:53.0) Gecko/20100101 Firefox/53.0")
opener = urllib2.build_opener()
opener.addheaders = [headers]

listurl = []

#thread1
class getURLThread(threading.Thread):
    def __init__(self,key,pagestart,pageend,proxy,urlqueue):
        threading.Thread.__init__(self)
        self.pagestart = pagestart
        self.pageend   = pageend
        self.proxy     = urlqueue
        self.key       = key
    def run(self):
        page = self.pagestart
        for page in range(self.pagestart,self.pageend+1):
            url='http://image.baidu.com/search/avatarjson?tn=resultjsonavatarnew&ie=utf-8&word=%E8%BE%B9%E7%89%A7&rn=60&pn='+str(60*page)
            data1 =opener.open(url).read()
            listurlpat = '"objURL":"(.+?)",'
            urlpage = re.compile(listurlpat,re.S).findall(data1)
            for urli in urlpage:
                time.sleep(7)
                for urlj in urlpage:
                    try:
                        print(urlj)
                        urlqueue.put(urlj)
                        urlqueue.task_done()
                    except urllib2.URLError as e:
                        if(hasattr(e,"code")):
                            print(e.code)
                        if(hasattr(e,"reason")):
                            print(e.reason)
                            time.sleep(10)
                    except Exception as e:
                        print('exception:'+str(e))
                        time.sleep(1)

class getContent(threading.Thread):
    def __init__(self,urlqueue,proxy):
        threading.Thread.__init__(self)
        self.urlqueue = urlqueue
        self.proxy = proxy
    def run(self):
        i = 1
        while(True):
            try:
                imagename = 'Baidu_Dog/'+str(i)+'.jpg'
                imageurl = urlqueue.get()
                urllib.urlretrieve(imageurl,imagename)
                print('get image'+url)
                i+=1
            except urllib2.URLError as e:
                if(hasattr(e,"code")):
                    print(e.code)
                if(hasattr(e,"reason")):
                    print(e.reason)
                    time.sleep(10)
            except Exception as e:
                print('exception:'+str(e))
                time.sleep(1)

class control(threading.Thread):
    def __init__(self,urlqueue):
        threading.Thread.__init__(self)
        self.urlqueue = urlqueue
    def run(self):
        while(True):
            print('process~ing')
            time.sleep(60)
            if(self.urlqueue.empty()):
                print('finished!')
                exit()

key='AI'
proxy = '119.6.136.122:80'
proxy2 = ''
pagestart = 1
pageend=40

t1 = getURLThread(key,pagestart,pageend,proxy,urlqueue)
t1.start()

t2 = getContent(urlqueue,proxy)
t2.start()

t3 = control(urlqueue)
t3.start()
```
上面的代码是用python2.7编写的，如果是再3.0的环境下需要进行一定的修改，不过也是很容易实现的，具体的实现我们就不说了，数据如何爬取我们在上一次总结过了，这一次只谈多线程，再程序中用了一个队列存取url，首先是html解析线程，这个线程主要的作用就是不停的解析html中的url然后存入队列中，另外一个线程就是数据保存线程，这个线程的主要作用就是将url中的图片保存下来，另外嗨哟偶一个线程，为什么需要有这个线程，因为再存取过程中我们并不直到哪个线程执行得更快，如果数据保存的线程执行的更快则有可能出现url中没有链接的情况，为了避免此时线程退出，我们用一个控制线程，控制如果url队列在60s都没有数据加入，则说明已经解析不到url了，则程序退出，通过控制线程对程序的退出条件进行了控制。
