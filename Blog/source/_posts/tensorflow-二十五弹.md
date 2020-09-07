---
title: tensorflow-二十五弹
date: 2018-03-05 23:55:02
tags: tensorflow学习
categories: 学习
---
忙活了几天终于整合完了，实现了手写数字识别的服务器版本！大家请撒花！！！！，废话不多说，直接上干货，实际上本章内容分为两个部分，首先是机器学习部分，这一部分在上一章中已经有说明，因此在本章中不会详细分析，第二个部分为服务器部分，服务器是使用的Python的Flask框架，主要思路过程为：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/%E6%9C%8D%E5%8A%A1%E5%99%A8%E8%AF%86%E5%88%AB%E6%A8%A1%E5%9E%8B.png">  
在这一章中主要讨论服务器建立的思路以及在识别的过程中的相关问题．实际上服务器的搭建比较简单，熟悉Flask框架的同学都能够很快的搭建一个服务器，但是其中有几个小问题需要注意，让我们先看代码：
```python
# -*- coding: utf-8 -*-
import os,base64,json,datetime
import datetime
from flask import Flask, request,render_template
from flask_uploads import UploadSet, configure_uploads, IMAGES
from CNN_Model import readModel
app = Flask(__name__)
m_predict = readModel.Predict()

@app.route('/', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        dataList = json.loads(request.data)
        imagedata = base64.b64decode(dataList['image'])
        date = datetime.datetime.now()
        datestr = date.strftime('%Y-%m-%d_%H-%M-%S')
        image_path = './ImageFiles/'+datestr+'.png'
        file=open(image_path,'wb')
        file.write(imagedata)
        file.close()
        full_path=os.getcwd()+'/ImageFiles/'+datestr+'.png'
        return str(m_predict.predict(full_path))
    return render_template('draw.html')


if __name__ == '__main__':
    app.run()
```
以上就是整个服务端的代码，代码主要就是一个图片上传，通过解析POST命令获取相关影像数据，实际上由于影像是绘制的，因此在获取过程中直接获取的是影像数据，然后将影像数据重命名为时间，并保存到本地，然后调用深度网络识别模块对影像进行识别并返回识别结果；虽然过程简单但是有几个坑是要注意的，在调用深度学习的识别过程中不能使用相对路径，需要使用绝对路径，因为调用深度学习的识别模块后相对路径可能出现错误，由此造成识别结果错误；另外返回值需要返回一个string类型的值，因此需要将数字的识别结果转换为string类型的字符,好了整个服务端的代码就介绍完毕了，下面是客户端的代码，主要分为两个部分，html文件以及js文件,在flask框架中需要将html文件保存到template文件夹中，将js文件保存到statics文件夹中，这是由于在渲染过程中，服务器会自动在这两个文件夹中进行搜索．代码为：

```html
<script src="draw.js"></script>
<script type="text/javascript">

</script>
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Document</title>
<style type="text/css">
    canvas{border:1px solid green;}
</style>
<script src="http://code.jquery.com/jquery-1.4.1.min.js"></script>
<script src="../static/draw.js"></script>
</head>
<body>
<div align="center">
    <canvas id="myCanvas" width="200" height="200" style="border:2px solid #6699cc"></canvas>
    <div class="control-ops">
	<form id="form1" runat="server">
    <button type="button" class="btn btn-primary" onclick="clearArea()">清空画板</button>
    Line width : <select id="selWidth">
		<option value="13">13</option>
		<option value="17">17</option>
		<option value="19">19</option>
		<option value="21">21</option>
		<option value="23">23</option>
		<option value="25">25</option>
    <option value="27">27</option>
    <option value="29">29</option>
    <option value="31">31</option>
    <option value="33" selected="selected">33</option>
    <option value="35">35</option>
    <option value="37">37</option>
    </select>
	<button type = "button" class="btn btn-primary" onclick="UploadPic()">识别</button>
	Result: <input type="text" name="result" id="recgResult" />
	</form>
    </div>
</div>
<script type="text/javascript">

</script>
</body>
</html>
```
实际上界面很简单，整个界面如图：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/%E6%95%B0%E5%AD%97%E8%AF%86%E5%88%AB%E6%9C%8D%E5%8A%A1%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%88%AA%E5%9B%BE.png">  
界面十分简单，一个绘制窗口的canvas一个清空按钮，一个识别按钮以及识别结果的输出框，实际上识别包括两个过程，绘图结果的上传和识别，实际上按钮调用的函数和画图函数都在js文件中：
```javascript
var mousePressed = false;
var lastX, lastY;
var ctx;

function InitThis() {
	  ctx= document.getElementById('myCanvas').getContext("2d");
		var w = ctx.canvas.width;
    var h = ctx.canvas.height;
		ctx.fillStyle = '#ffffff';
		ctx.fillRect(0,0,w,h);

    $('#myCanvas').mousedown(function (e) {
        mousePressed = true;
        Draw(e.pageX - $(this).offset().left, e.pageY - $(this).offset().top, false);
    });

    $('#myCanvas').mousemove(function (e) {
        if (mousePressed) {
            Draw(e.pageX - $(this).offset().left, e.pageY - $(this).offset().top, true);
        }
    });

    $('#myCanvas').mouseup(function (e) {
        mousePressed = false;
    });
    $('#myCanvas').mouseleave(function (e) {
        mousePressed = false;
    });
}

function Draw(x, y, isDown) {
    if (isDown) {
        ctx.beginPath();
        ctx.strokeStyle = $('#selColor').val();
        ctx.lineWidth = $('#selWidth').val();
        ctx.lineJoin = "round";
        ctx.moveTo(lastX, lastY);
        ctx.lineTo(x, y);
        ctx.closePath();
        ctx.stroke();
    }
    lastX = x; lastY = y;
}

function clearArea() {
    // Use the identity matrix while clearing the canvas
    ctx.setTransform(1, 0, 0, 1, 0, 0);
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
		var w = ctx.canvas.width;
		var h = ctx.canvas.height;
		ctx.fillStyle = '#ffffff';
		ctx.fillRect(0,0,w,h);
}

function UploadPic(){
	var Pic = document.getElementById("myCanvas").toDataURL("image/png");
    Pic = Pic.replace(/^data:image\/(png|jpg);base64,/, "")
	$.ajax({
		type:'POST',
		url:'',
		data:'{"image":"'+Pic+'"}',
		contentType:'application/json;charset=utf-8',
		dataType:'json',
		success:function(msg){
			$('#recgResult').val(msg);
		}
	});

}

window.onload = function (){
	InitThis();
}
```
js文件包含了四个函数，分别为初始化函数，绘制函数，清空画图版函数和图片上传函数，在这里着重说一下图片上传，在js中是通过ajax构造request参数的方式将图片数据转换为数据串然后上传到服务器上的，另外有一个需要注意的点canvas保存的是png影像，会有一个透明度，透明的png会造成较大的识别误差，因此需要将背景绘制为白色；到此整个服务器端识别过程就结束了．由于上一章详细说明了神经网络的构建，在这里就不进行详细说明，有一个点需要注意，在训练过程中所使用的mnist数据归一化到了0-1之间，且进行了取反操作，因此我们在读取图片后不要忘记对图片进行归一化和取反，否则会造成图像识别结果出现较大误差．
识别结果如图：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/%E6%95%B0%E5%AD%97%E8%AF%86%E5%88%AB%E6%9C%8D%E5%8A%A1%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%88%AA%E5%9B%BE.png">  
整个项目上传在github中，有兴趣可以在[我的github主页](https://github.com/RemoteSensingFrank)上查看
