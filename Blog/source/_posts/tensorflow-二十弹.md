---
title: tensorflow-二十弹
date: 2017-12-17 10:35:05
tags: tensorflow学习
categories: 学习
---
这一篇本来不算是tensorflow的学习，不过也是用python写的一个爬虫小程序，因此可以归于这一类吧，毕竟获取数据也是深度学习的一个重要的方面．主要原因是刘艳前几天突然问我能不能帮她爬一个医药网站上的数据，我粗略的看了一下那个网站，都是以表格形式获取的数据，整个网站的数据都在＇\<tr> \<td>＇标签之间，看起来数据很好获取的样子，作为一个技术宅我当然想都没有想就说OK啦，毕竟这样的网站一般不会做太高级的反爬措施，所以数据获取相对来说也比较容易．  
因此下班回家之后我就开始分析网站了，我们截取网站的源代码看看：
```html
<caption align="Left">
			---------国产药品数据区（点击列名可按该项排序）---------
		</caption><tr style="color:White;background-color:#000084;font-weight:bold;">
			<th scope="col">序号</th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$ApprovalNumber&#39;)" style="color:White;">批准文号</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$ApprovalNumberOld&#39;)" style="color:White;">原批准文号</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$NDCNumber&#39;)" style="color:White;">药品本位码</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$NDCNumberRemark&#39;)" style="color:White;">本位码注释</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$ProductName&#39;)" style="color:White;">产品名称</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$EnglishName&#39;)" style="color:White;">英文名称</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$TradeName&#39;)" style="color:White;">商品名</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$Manufacturer&#39;)" style="color:White;">生产厂家</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$AddressOfManufacturer&#39;)" style="color:White;">生产地址</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$Dosage&#39;)" style="color:White;">规格</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$Form&#39;)" style="color:White;">剂型</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$Category&#39;)" style="color:White;">类别</a></th><th scope="col"><a href="javascript:__doPostBack(&#39;GridViewNational&#39;,&#39;Sort$ApprovalDate&#39;)" style="color:White;">批准日期</a></th>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>1</td><td>国药准字Z20063394</td><td>&nbsp;</td><td>86903909000440</td><td>&nbsp;</td><td>明目十六味丸</td><td>&nbsp;</td><td>&nbsp;</td><td>内蒙古蒙药股份有限公司</td><td>通辽经济技术开发区辽河大街西段</td><td>每10丸重2g</td><td>丸剂（水丸）</td><td>中药</td><td>2016-04-12</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>2</td><td>国药准字Z20110009</td><td>&nbsp;</td><td>86900535000233</td><td>&nbsp;</td><td>康肾丸</td><td>&nbsp;</td><td>&nbsp;</td><td>深圳市泰康制药有限公司</td><td>深圳市宝安区观澜镇大水坑新塘泰康工业园</td><td>每78粒重6g</td><td>丸剂</td><td>中药</td><td>2016-03-14</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>3</td><td>国药准字Z20163015</td><td>国药准字Z42021015</td><td>86904123000865</td><td>&nbsp;</td><td>八珍益母丸</td><td>----</td><td>&nbsp;</td><td>华润三九(临清)药业有限公司</td><td>山东省临清市北关街19号</td><td>----</td><td>丸剂</td><td>中药</td><td>2016-03-04</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>4</td><td>国药准字Z36020239</td><td>&nbsp;</td><td>86905290000450</td><td>&nbsp;</td><td>清热安宫丸</td><td>&nbsp;</td><td>&nbsp;</td><td>江西国药有限责任公司</td><td>江西省南昌市小蓝工业园国药大道888号</td><td>每15丸重2.0g</td><td>丸剂(水蜜丸)</td><td>中药</td><td>2016-02-26</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>5</td><td>国药准字Z20063933</td><td>&nbsp;</td><td>86905778000392</td><td>&nbsp;</td><td>六味地黄丸</td><td>&nbsp;</td><td>&nbsp;</td><td>海南伊顺药业有限公司</td><td>海南省海口市桂林洋开发区</td><td>每袋装1.7g(相当于原药材3g)</td><td>丸剂</td><td>中药</td><td>2016-02-22</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>6</td><td>国药准字Z36021158</td><td>&nbsp;</td><td>86905325000486</td><td>&nbsp;</td><td>田七跌打丸</td><td>&nbsp;</td><td>&nbsp;</td><td>江西民济药业有限公司</td><td>江西省铜鼓县迎宾大道168号</td><td>每丸重6g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-02-16</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>7</td><td>国药准字Z20063338</td><td>&nbsp;</td><td>86904438000871</td><td>&nbsp;</td><td>左归丸</td><td>&nbsp;</td><td>&nbsp;</td><td>上海华源制药安徽广生药业有限公司</td><td>安徽省界首市华源大道6号</td><td>每10丸重1g</td><td>丸剂(水蜜丸)</td><td>中药</td><td>2016-02-16</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>8</td><td>国药准字Z20093584</td><td>&nbsp;</td><td>86902573000565</td><td>&nbsp;</td><td>六味地黄丸(浓缩丸)</td><td>&nbsp;</td><td>&nbsp;</td><td>承德燕峰药业有限责任公司</td><td>承德市宽城满族自治县民族街280号</td><td>每8丸重1.44g(每8丸相当于原药材3g)</td><td>丸剂(浓缩丸)</td><td>中药</td><td>2016-02-01</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>9</td><td>国药准字Z20093409</td><td>&nbsp;</td><td>86902573000534</td><td>&nbsp;</td><td>归脾丸</td><td>&nbsp;</td><td>&nbsp;</td><td>承德燕峰药业有限责任公司</td><td>承德市宽城满族自治县民族街280号</td><td>每8丸重1.36g(每8丸相当于原生药3g)</td><td>丸剂(浓缩丸)</td><td>中药</td><td>2016-02-01</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>10</td><td>国药准字H22023097</td><td>&nbsp;</td><td>86903372000688</td><td>&nbsp;</td><td>维生素B1丸</td><td>Vitamin B1 Pills</td><td>&nbsp;</td><td>吉林金恒制药股份有限公司</td><td>吉林省吉林市吉林经济技术开发区人达街9号</td><td>10mg</td><td>丸剂</td><td>化学药品</td><td>2016-01-29</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>11</td><td>国药准字Z20064145</td><td>&nbsp;</td><td>86900200000353</td><td>&nbsp;</td><td>大黄?虫丸</td><td>&nbsp;</td><td>&nbsp;</td><td>北京亚东生物制药有限公司</td><td>北京市大兴区生物医药产业基地祥瑞大街29号</td><td>每60丸重3g</td><td>丸剂(水蜜丸)</td><td>中药</td><td>2016-01-26</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>12</td><td>国药准字Z20063573</td><td>&nbsp;</td><td>86900169000234</td><td>&nbsp;</td><td>疏风定痛丸</td><td>&nbsp;</td><td>&nbsp;</td><td>北京同仁堂科技发展股份有限公司制药厂</td><td>北京市北京经济技术开发区东环北路5号</td><td>每100丸重20g</td><td>丸剂(水蜜丸)</td><td>中药</td><td>2016-01-25</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>13</td><td>国药准字Z20063839</td><td>&nbsp;</td><td>86903485001114</td><td>&nbsp;</td><td>鱼鳔补肾丸</td><td>&nbsp;</td><td>&nbsp;</td><td>钓鱼台医药集团吉林天强制药股份有限公司</td><td>吉林省柳河县人民大街309号</td><td>每10丸重2.9g(相当于原生药2.7g)</td><td>丸剂(水丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>14</td><td>国药准字Z22020445</td><td>&nbsp;</td><td>86903285001413</td><td>&nbsp;</td><td>肥儿丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重3g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>15</td><td>国药准字Z22020516</td><td>&nbsp;</td><td>86903285000607</td><td>&nbsp;</td><td>牛黄上清丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重6g</td><td>丸剂</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>16</td><td>国药准字Z22020517</td><td>&nbsp;</td><td>86903285000669</td><td>&nbsp;</td><td>杞菊地黄丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重9g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>17</td><td>国药准字Z20025012</td><td>&nbsp;</td><td>86903285000225</td><td>&nbsp;</td><td>牛黄清脑开窍丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重6g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>18</td><td>国药准字Z20063513</td><td>&nbsp;</td><td>86903485000094</td><td>&nbsp;</td><td>骨筋丸胶囊</td><td>&nbsp;</td><td>&nbsp;</td><td>钓鱼台医药集团吉林天强制药股份有限公司</td><td>吉林省柳河县人民大街309号</td><td>每粒装0.3g</td><td>胶囊剂</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>19</td><td>国药准字Z22020432</td><td>&nbsp;</td><td>86903285001444</td><td>&nbsp;</td><td>回天再造丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重10g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>20</td><td>国药准字Z22020599</td><td>&nbsp;</td><td>86903285000072</td><td>&nbsp;</td><td>八珍益母丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>----</td><td>丸剂(小蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>21</td><td>国药准字Z22020653</td><td>&nbsp;</td><td>86903285000027</td><td>&nbsp;</td><td>安宫牛黄丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重3g</td><td>丸剂</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>22</td><td>国药准字Z22020672</td><td>&nbsp;</td><td>86903285000782</td><td>&nbsp;</td><td>天王补心丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>----</td><td>丸剂(小蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>23</td><td>国药准字Z22020651</td><td>&nbsp;</td><td>86903285000867</td><td>&nbsp;</td><td>壮腰健肾丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重5.6g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>24</td><td>国药准字Z22020662</td><td>&nbsp;</td><td>86903285001093</td><td>&nbsp;</td><td>高丽清心丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重7g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>25</td><td>国药准字Z22020664</td><td>&nbsp;</td><td>86903285001000</td><td>&nbsp;</td><td>黄连羊肝丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重9g</td><td>丸剂</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>26</td><td>国药准字Z22020602</td><td>&nbsp;</td><td>86903285001529</td><td>&nbsp;</td><td>补中益气丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>----</td><td>丸剂(水丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>27</td><td>国药准字Z22020586</td><td>&nbsp;</td><td>86903285000348</td><td>&nbsp;</td><td>乌鸡白凤丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重9g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>28</td><td>国药准字Z22020443</td><td>&nbsp;</td><td>86903285000263</td><td>&nbsp;</td><td>香砂养胃丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>----</td><td>丸剂(水丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:#EEEEEE;">
			<td>29</td><td>国药准字Z22020435</td><td>&nbsp;</td><td>86903285001031</td><td>&nbsp;</td><td>大活络丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>每丸重3.5g</td><td>丸剂(大蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr style="color:Black;background-color:Gainsboro;">
			<td>30</td><td>国药准字Z22020518</td><td>&nbsp;</td><td>86903285000409</td><td>&nbsp;</td><td>杞菊地黄丸</td><td>&nbsp;</td><td>&nbsp;</td><td>吉林四环澳康药业有限公司</td><td>吉林省龙井工业集中区</td><td>----</td><td>丸剂(水蜜丸)</td><td>中药</td><td>2016-01-19</td>
		</tr><tr align="center" style="color:Black;background-color:#999999;">
```
查看源码发现其实很容易爬取数据的，可是如何获取下一页呢，这个问题把我难住了，因为以前获取的网站的下一页都是通过url来体现的，而这一个网站在翻页的过程中并没有url的变化，因此我很奇怪，为什么没有url的变化，那是通过什么方式翻页呢，所以我开始对网页的请求一个一个的进行分析，后来我在请求头中看到提交了一个表单，情况如下：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/form%E7%88%AC%E8%99%AB.png"/>  
通过上面这张图我们可以看到，不同页面的变化就是Page$的变化，所以我们可以根据这个进行翻页，我们了解到翻页就是提交表单，因此在翻页的过程中我们只需要模拟提交表单就行了，从图中我们看出表单中存在一个属性为viewstate这个属性控制的是显示模式，不需要管，直接拷贝过来就好了．因此我们可以构建一个模拟表单提交的形式，代码表示如下：  
```python
import sys
reload(sys)
sys.setdefaultencoding('utf8')
import requests
import re
import urllib
import urllib2
f = open('test.txt','w')
for num in range(1,3):
    strpage='Page$'+str(num)
    keywords={
    '__EVENTTARGET':'GridViewNational',
    '__EVENTARGUMENT':strpage,
    '__LASTFOCUS':'',
    '__VIEWSTATE':'xxx'
    }

    url="http://www.drugfuture.com/cndrug/search.aspx?SearchTerm=%u4e38&DataFieldSelected=auto"
    headers=("User-Agent","Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:53.0) Gecko/20100101 Firefox/53.0")
    opener = urllib2.build_opener()
    req = urllib2.Request(url, headers)
    keywords = urllib.urlencode(keywords)
    html1 = opener.open(url,keywords).read()
    res_tr1 = r'id="GridViewNational" (.*?)<table'
    res_tr2 = r'<tr(.*?)/tr>'
    res_tr3 = r'<font color="Black">(.*?)</font>'
    m_tr1 =  re.findall(res_tr1,html1,re.S|re.M)
    for s in m_tr1:
        m_tr2 =  re.findall(res_tr2,s,re.S|re.M)
        for s2 in m_tr2:
            m_tr3 =  re.findall(res_tr3,s2,re.S|re.M)
            line=""
            for s3 in m_tr3:
                line=line+","+s3
            print >> f, "%s" %(line)
f.close()
```
具体的表单我们就不写了，反正拷过来就行了，我们来分析一下代码，主要的过程就是提交表单的过程，这个过程理解了实际上也就没有什么问题了．首先引入模块，然后构造循环，我们可以看到，构造一个提交表单，然后在每次获取新的页面之后按照同样的方式解析并输出到文件中就可以了．
