---
title: ArcGIS开发相关技术总结1
date: 2019-01-18 10:39:09
tags: ArcGIS开发
categories: 学习
---
&nbsp;&nbsp;&nbsp;&nbsp;专心做了两个月的GIS开发，使用的是ArcGIS的平台，遇到了很多问题，对ArcGIS的API的使用有了更多的体会，另外对于js前端开发也有了更深刻的体会我将就最近做的这个项目集中总结一下在使用ArcGIS API的过程中所遇到的问题以及解决方式：

* 问题1 Feature查询
  
&nbsp;&nbsp;&nbsp;&nbsp;这个问题是一个很基础的问题，但是一直没有引起我的重视，直到这次进行开发才重视起来，实际上查询查询很有意思，主要有两种方式，实际上两种方式都差不多，之所以要提两种方式是因为并不是所有人都会直接使用ArcGIS js API对接ArcGIS Enterprise的平台，也许有人用OpenLayer有人用Cesium，在使用这些平台的时候我们所提到的第二种方式就很有意义了；第一种方式是直接通过FeatureLayer的Query函数进行查询，具体实现方式为：  
```javascript
/**
    * 根据查询语句进行查询
    * @param {*} whereStatment：查询语句
    */
queryTask:function(layer,whereStatment){
    var query = layer.createQuery();
    query.outFields=["*"];
    query.where=whereStatment;
    return layer.queryFeatures(query);
},

/**
    * 根据范围进行查询
    * @param {*} pt:几何信息
    */
queryGeometryTask:function(layer,pt,whereStatment){
    var query = layer.createQuery();
    query.outFields=["*"];
    query.where=whereStatment;
    query.geometry=pt;
    return layer.queryFeatures(query);
},

/**
    * 根据范围where语句和字段进行查询
    */
queryGeometryTaskFiled:function(layer,pt,whereStatment,field){
    var query = layer.createQuery();
    query.outFields=[field];
    query.where=whereStatment;
    query.geometry=pt;
    return layer.queryFeatures(query);
}
```
以上是几个直接利用FeatureLayer进行查询的代码封装，实际上就是利用**createQuery**函数创建一个查询对象然后给查询对象查询条件然后进行查询。这个没有什么好说的，如果想要进一步的了解的话可以查询[官网]("https://developers.arcgis.com/javascript/latest/api-reference/esri-layers-FeatureLayer.html")的说明。  
&nbsp;&nbsp;&nbsp;&nbsp;下面我们来聊聊第二种查询方式，直接通过RESTAPI来查，相比于上面集成进ArcGIS js API中的查询方式，通过RESTAPI的查询方式要复杂很多，实际上对于一个发布的数据，我们进入它的服务地址，然后点击query可以看到很多查询条件实际上查询出来的结果是一个soap的结果，为了得到json数据我们在查询的时候需要添加上&f=json这个查询条件，通过这个查询条件可以查询到json格式的数据包括属性和位置信息，然后拿到这个数据后自己构建需要展示的数据，对于不同的平台和API展示数据的构建形式都不同，在这里就不一一进行演示了。虽然这样的方式相对比较复杂，但是具有更加广泛的适用性，如果以后适用不太API进行开发这也不失为一种方案。

* 问题2 三维实体的查询

&nbsp;&nbsp;&nbsp;&nbsp;这个问题是一个很有代表性的问题，在三维需要日益发展的今天GIS的呈现方式也由过去的二维为主发展到目前的三维为主，而ArcGIS中也有了SceneLayer这样一个展示三维数据的图层。一切看似很美好，但是有一朵乌云总是悬在头顶，那就是SceneLayer的查询。  
**queryExtent()：**	Promise<Object>	Executes a Query against the service and returns the 2D Extent of features that satisfy the query.more details	SceneLayer  
**queryFeatureCount()：**	Promise<Number>	Executes a Query against the service and returns the number of the features that satisfy the query.more details	SceneLayer  
**queryFeatures()：**	Promise<FeatureSet>	Executes a Query against the service and returns a FeatureSet.more details  
上面三个函数式SceneLayer有关查询的函数，看似好像都差不多实际上与FeatureLayer的查询功能相比SceneLayer的查询功能弱了很多，从官网的细节介绍可以看到：<font color=#ff0000>**The query succeeds only if the layer's supportsLayerQuery capability is enabled.**</font> 这句话很重要，它说需要支持LayerQuery才能拿进行查询，而如果才能够支持呢，需要在发布的时候link一个FeatureLayer，如何link呢需要通过Pro去发布，如果直接导出为SLPK的话是没有的，这个是社区的[ESRI技术人员的回答](https://community.esri.com/message/645017-re-js-api-scene-layer-queryfeatures-throw-scenelayerquery-not-available-error?commentID=645017#comment-645017)，关于这个问题我向ESRI的技术支持也询问过，他们似乎也不知道具体怎么弄，所以查询我就放弃采用这种方式了，但是看官网的例子可以看出如果数据是合理的应该是可以查询到的；这种方式不行就换一个方式，直接通过SceneLayerView进行查询，这样查询很好但是必须少于10000个要素，实际上同时展示的要素大于10000个的情况也不会太多，这个时候我采用的做法是先zoomIn，减少视野内要素个数，然后进行查询，具体的操作代码为：
```javascript
/**
    * 从图层呢个veiw中搜索出相关建筑实体并高亮展示
*/
searchHighLightBuildingTransIDs:function(layer,transIds){
    var query = new Query();
    query.outFields=["*"];
    this.view.whenLayerView(layer).then(function(layerView){
            layerView.queryFeatures(query).then(function(response){
                    var graphic = response.features.filter(function(hgraphic){
                            var i=0;
                            for(i=0;i<transIds.length;++i){
                                    if(hgraphic.attributes.INNER_TRAN==transIds[i]){
                                            return hgraphic.attributes.INNER_TRAN==transIds[i];
                                    }
                                    else{
                                            continue;
                                    }
                            }
                    });
                    if(this.highlight){
                        this.highlight.remove();
                    }
                    this.highlight=layerView.highlight(graphic);
            });
    })
},
```
以上代码是查询并高亮展示要素的核心代码。

* 问题3 关于非公开链接的登录问题  

&nbsp;&nbsp;&nbsp;&nbsp;老实说关于这个问题我是很不愿意去说的，因为<font color=#ff0000>**不安全**</font>但是有了这个需求，所以不得不去实现，但是我还是要强调，既然是是公开资源就干脆公开，如果需要设计权限那么直接设置就好了或者能不能通过OAuth2进行控制，这个也在研究，但是最简单的方式就是目前这个方式，通过identityManager来实现，具体的方式为：
```javascript
require([
      "esri/Map",
      "esri/views/MapView",
	  "esri/layers/FeatureLayer",
	  "esri/identity/ServerInfo",
	  "esri/identity/IdentityManager"
    ], function(Map, MapView,FeatureLayer,ServerInfo,esriId) {

      var map = new Map({
        basemap: "streets"
      });

      var view = new MapView({
        container: "viewDiv",
        map: map,
        zoom: 4,
        center: [15, 65] // longitude, latitude
      });
	  var svrInfo = new ServerInfo();
	  svrInfo.serverString 	  = "https://sampleserver6.arcgisonline.com/arcgis/rest/services";
	  svrInfo.tokenServiceUrl = "https://sampleserver6.arcgisonline.com/arcgis/tokens/";
	  var userInfo = {username:"user1",password:"user1"};
	  
	  esriId.generateToken(svrInfo,userInfo).then(function(data){
		var tokenVal = data.token;
		console.log(data);
		esriId.registerToken({server:"https://sampleserver6.arcgisonline.com/arcgis/rest/services",token:tokenVal});
	  });
	  
      var secureLayer = new FeatureLayer({
		url:"https://sampleserver6.arcgisonline.com/arcgis/rest/services/SaveTheBay/FeatureServer/0",
      });
      map.add(secureLayer);
    });
```
&nbsp;&nbsp;&nbsp;&nbsp;上面代码是一个官网的例子我改造的结果，通过generateToken生成token然后将token注册到我们需要访问的服务器路径，这样在访问的时候就会带上token，实际上是一个很简单的过程，主要是这几个路径的设置。

* 问题4 炫酷的三维界面展示 

&nbsp;&nbsp;&nbsp;&nbsp;这个问题很抽象，什么样的界面才能被称为酷炫？这个是业主提出来的一个需求，主要是看到了一些示例代码，然后看到一些特别酷炫的效果的时候就希望我们也能够做出酷炫的效果，实际上一开始我也不知道具体应该怎么做出酷炫的效果，后来发现实际上酷炫的效果就是一些光流效果，呼吸灯效果以及跑马灯的效果等，这些效果实际上并不算难做，但是结合三维球就不知道应该怎么去做了，不过在研究中我还是领悟了一种方法，利用ArcGIS JS API的MESH来实现，实际上这个mesh是一个几何结构，我理解为网格类似于点线面的一个结构，然后构成了一个三角网，实际上这个功能在示例代码中是用来做虚拟地形模型的，但是根据我的脑洞我觉得可以有更加酷炫的应用，所以针对这个结构进行了一下深入了解。
```javascript
/**
*  搜索出变压器一定范围内的变压器然后渲染并渲染为三维数据进行展示 
*/
RenderTransformers:function(layer,pnt,range){
    //首先根据点击点的位置搜索一定范围内的变压器数目
    const exaggeration = 2;
    var points=[];

    for(let index1=0;index1<50;++index1){
        for(let index2 = 0 ;index2<50;++index2){
            try{
                points.push([pnt.x-750+30*index1,pnt.y-750+30*index2]);
            }catch(error){
                console.log(error);
            }
        }
    }


    //获取变压器功率进行插值
    var query = layer.createQuery();
    query.outFields=["*"];
    var extentRange = new Extent({
        xmax:pnt.x+750,
        xmin:pnt.x-750,
        ymax:pnt.y+750,
        ymin:pnt.y+750,
        SpatialReference:SpatialReference.WebMercator
    }); 
    query.geometry=extentRange;

    var mpnts= new MultiPoint(points);
    mpnts.spatialReference=SpatialReference.WebMercator;
    _this=this;
    layer.queryFeatures(query).then(function(result){
        var featurePower=result.features.map(function(graphic){
            if(graphic.geometry.type=="point"){
                return {"x":pnCent.x,"y":pnCent.y,"z":graphic.attributes["P_S_AVG_1"]}
            }
        });
        var mesh = new Mesh({spatialReference:SpatialReference.WebMercator});
        var data=[];

        //强行写一个插值函数出来
        //TODO:
    });

    var mpnts= new MultiPoint(points);
    mpnts.spatialReference=SpatialReference.WebMercator;
    _this=this;
    try{
        layer.queryElevation(mpnts).then(function(result){
            var mesh = new Mesh({spatialReference:SpatialReference.WebMercator});
            var data=[];
            var tPnts=result.geometry.points;
            for(let i=0;i<50;++i){
                for(let j=0;j<50;++j){
                    tPnts[i*50+j][2]=tPnts[i*50+j][2]+200;
                }
            }
            for(let i=0;i<50-1;i++){
                for(let j = 0 ;j<50-1;j++){
                    data.push(tPnts[i*50+j][0]);data.push(tPnts[i*50+j][1]);data.push(tPnts[i*50+j][2]);
                    data.push(tPnts[(i+1)*50+j][0]);data.push(tPnts[(i+1)*50+j][1]);data.push(tPnts[(i+1)*50+j][2]);
                    data.push(tPnts[i*50+(j+1)][0]);data.push(tPnts[i*50+(j+1)][1]);data.push(tPnts[i*50+(j+1)][2]);

                    data.push(tPnts[(i+1)*50+(j+1)][0]);data.push(tPnts[(i+1)*50+(j+1)][1]);data.push(tPnts[(i+1)*50+(j+1)][2]);
                    data.push(tPnts[(i+1)*50+j][0]);data.push(tPnts[(i+1)*50+j][1]);data.push(tPnts[(i+1)*50+j][2]);
                    data.push(tPnts[i*50+(j+1)][0]);data.push(tPnts[i*50+(j+1)][1]);data.push(tPnts[i*50+(j+1)][2]);
                }
            }
            mesh.vertexAttributes.position=data;
            var component=new MeshComponent({
                material:{
                    color:_this.RenderPowerColor(mesh)
                },
                shading:"flat"
            })
            mesh.components=[component];
            if(_this.IsOrNot){
                var graphic = new Graphic({
                    geometry:mesh,
                    symbol:{
                        type:"mesh-3d",
                        symbolLayers:[{
                            type:"fill",
                        }],
                    },

                    attributes:{"name":"graphic1"},

                });
                _this.coolTransGraphicLayer.add(graphic);
                _this.coolTransGraphicLayer.removeMany(_this.coolTransGraphicLayer.graphics.map(function(graphic){
                    return graphic.attributes['name']=='graphic2'?graphic:null;
                }));
                _this.IsOrNot=false;
            }else{
                var graphic = new Graphic({
                    geometry:mesh,
                    symbol:{
                        type:"mesh-3d",
                        symbolLayers:[{
                            type:"fill"
                        }]
                    },
                    matreial:{
                        color:[255,0,0,1]
                    },
                    attributes:{"name":"graphic2"},
                });
                _this.coolTransGraphicLayer.add(graphic);
                _this.coolTransGraphicLayer.removeMany(_this.coolTransGraphicLayer.graphics.map(function(graphic){
                    return graphic.attributes['name']=='graphic1'?graphic:null;
                }));
                _this.IsOrNot=true;
            }
        }).catch(function(error){
            console.log(error);
        });
    }catch(error){
        console.log(error);
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;针对以上代码进行分析，首先构建网格，实际上就是网格点，在以上应用中我是根据输入的点给一个区域范围然后按照一定的比例采样得到一个50*50的格网，通过网格确定了每一个点的经纬度坐标，实际上可以是投影坐标。确定了坐标之后就要渲染了，然后根据要渲染的字段查出要渲染的强度，实际上为了让其显示在地面上我是根据地面的高度加了一个值实际上也可以直接给一个值，这个不重要，这样构造一个mesh的数据，构造数据的时候一定要注意mesh是一个三维网，所以构造的时候一定要三个点构成一个面三个点构成一个面，注意点的顺序不要乱，然后像普通渲染graphic一样渲染这个mesh的结构就可以了。