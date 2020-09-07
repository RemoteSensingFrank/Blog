---
title: ArcGIS绘图功能
date: 2019-12-03 19:32:00
tags: ArcGIS开发
categories: 学习
---
### 概述：
&nbsp; &nbsp; &nbsp; &nbsp;在GIS应用中有一个十分重要的应用为绘图应用，一直以来基于Web的GIS在绘图方面的应用都是弱项，这其中主要问题在于Web GIS在应用过程中，所有数据都是通过浏览器进行存储和渲染，而浏览器所能够使用到的计算机资源是有限的，因此相比于给予客户端的GIS软件，基于Web GIS的应用中绘制的功能较弱，因此Web GIS更偏向于数据展示，然而随着计算机性能的发展与WebGL技术的发展，Web GIS的应用领域得到了极大的扩展，以前看起来较为复杂的绘制功能实际上也能够在浏览器端得到比较好的实现，下面我将仔细描述一下通过ArcGIS进行WebGIS应用的绘图功能。
### ArcGIS绘图功能说明：

&nbsp; &nbsp; &nbsp; &nbsp;基于ArcGIS的绘图功能的实现实际上主要是通过Draw这个对象来实现的，在 [官网](https://developers.arcgis.com/javascript/latest/api-reference/esri-views-draw-Draw.html)上会有比较详细的Demo和说明，但是为了方便理解以及应用，我这里还是进行一些简单的总结：
* 1 通过Sketch来实现绘图功能，实际上简单的来说就是指直接通过Sketch Widget这个控件来实现绘图功能；
* 2 通过Draw函数实现绘图功能；

&nbsp; &nbsp; &nbsp; &nbsp;如果想要采用第一种方法进行绘制，则方法很简单，引入对应的控件，然后就能够实现画图功能，包括绘制点，线，面以及一些包括正方形，圆形在内的规则形状，同时还提供了选择、编辑以及删除绘制的图形等功能，通过此控件进行绘制主要优点在于使用简单，能够快速的搭建起一个绘制的功能，但是通过此控件进行绘图也存在着一些问题，主要问题在于无法进行定制化的应用，如果只是需要一个简单而且功能全面的绘图控件，则此控件完全符合要求，如果功能要求有较强的定制化，则此控件可能难以满足要求，另外对于完成的控件的定制化开发可能需要花费更多的时间，如果大家想要了解Sketch控件的使用，直接关注[ArcGIS的Demo网站](https://developers.arcgis.com/javascript/latest/sample-code/sketch-geometries/index.html)查看示例代码就可以了。

&nbsp; &nbsp; &nbsp; &nbsp;对于大部分应用来说，直接使用空间远远满足不了应用要求，需要提供更加底层的API进行定制化的开发，这样我们就需要使用到Draw模块了。实际上这个模块的使用过程也比较简单，主要包括以下几个步骤：1.引入Draw模块；2.Draw模块与view视图进行绑定；3.定义绘制的方法；4.绘制方法d的响应；下面我们针对以上几个步骤进行分别的说明：

&nbsp; &nbsp; &nbsp; &nbsp;1.引入绘制函数：“esri/views/draw/Draw”，直接通过require模块引入绘制函数模块，然后直接定义全局绘制函数绑定view以便于进行绘制；2.Draw模块与View进行绑定，定义绘制方法：
```
 // create a new instance of draw
var draw = new Draw({
  view: view
});

// create an instance of draw polyline action
// the polyline vertices will be only added when
// the pointer is clicked on the view
var action = draw.create("polyline", {mode: "click"});

// fires when a vertex is added
action.on("vertex-add", function (evt) {
  measureLine(evt.vertices);
});

// fires when the pointer moves
action.on("cursor-update", function (evt) {
  measureLine(evt.vertices);
});

// fires when the drawing is completed
action.on("draw-complete", function (evt) {
  measureLine(evt.vertices);
});

// fires when a vertex is removed
action.on("vertex-remove", function (evt) {
  measureLine(evt.vertices);
});

function measureLine(vertices) {
  view.graphics.removeAll();

  var line = createLine(vertices);
  var lineLength = geometryEngine.geodesicLength(line, "miles");
  var graphic = createGraphic(line);
  view.graphics.add(graphic);
}
```
通过action定义绘制方法，在ArcGIS中的绘制方法包括：Point，MutiPoint，Polyline，Polygon，定义绘制方法之后就需要响应绘制函数，实际上对于绘制不同的地物，对应需要响应的方法也存在差异，如果需要将绘制的结果进行保存，一般l来说我们再新建一个GraphicsLayer图层，通过此图层进行保存，将数据保存到图层中。
整体代码为：

```javascript
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta
      name="viewport"
      content="initial-scale=1,maximum-scale=1,user-scalable=no"
    />
    <title>Draw polyline - 4.13</title>

    <link
      rel="stylesheet"
      href="https://js.arcgis.com/4.13/esri/themes/light/main.css"
    />
    <script src="https://js.arcgis.com/4.13/"></script>
    <script src="http://libs.baidu.com/jquery/2.1.4/jquery.min.js"></script>
    <style>
      html,
      body,
      #viewDiv {
        height: 100%;
        width: 100%;
        margin: 0;
        padding: 0;
      }
    </style>

    <script>
      require([
        "esri/Map",
        "esri/views/MapView",
        "esri/views/draw/Draw",
        "esri/Graphic",
        "esri/layers/GraphicsLayer",
        "esri/geometry/geometryEngine"
      ], function(Map, MapView, Draw, Graphic, GraphicsLayer,geometryEngine) {
        const map = new Map({
          basemap: "gray"
        });

        const view = new MapView({
          container: "viewDiv",
          map: map,
          zoom: 16,
          center: [18.06, 59.34]
        });
        //var layerSave = new GraphicsLayer();
        var layerSavePoint = new GraphicsLayer();
        //map.add(layerSave);
        map.add(layerSavePoint);
        var param={
              "devType":0,
              "schemeId":"07b0486b0e014b5ba80685d72dc38351"
            };
        $.ajax({
              url: "http://192.168.43.214:8087/visual/query-project-scheme-dev",
              type: "post",
              data: JSON.stringify(param),
              headers: {
                  'Origin':"*",
                  'content-type': "application/json",
                  // 'token': "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NzQ5MjUzNzAsInVzZXJuYW1lIjoiaHowMDEifQ.P1Rzs0A6klHU6CZqBuItibpya-kfsT5kJ1w7EboJtAo"
              },
              success: function(result) {  //这里就是我出错的地方
                console.log(result.data.list[0]);
                for(var i=0;i<result.data.list.length;++i){
                  var graphic=Graphic.fromJSON(JSON.parse(result.data.list[i]["geometryInfo"]));
                  layerSavePoint.graphics.add(graphic);
                }
              },
              error: function(data) {
                console.log("请求出错");
              }
        });

        // add the button for the draw tool
        view.ui.add("line-button", "top-left");

        const draw = new Draw({
          view: view
        });

        var drawType=0;

        // draw polyline button
        document.getElementById("line-button").onclick = function() {
            const action = draw.create("polyline");
            action.on(
              [
                "vertex-add",
                "vertex-remove",
                "cursor-update",
                "redo",
                "undo"
              ],
              updateVertices
            );  
            action.on("draw-complete", function (evt) {
              saveLineGraphic(evt);
            });
        };

        // Checks if the last vertex is making the line intersect itself.
        function updateVertices(event) {
          // create a polyline from returned vertices
          if (event.vertices.length > 1) {
            const result = createLineGraphic(event);
          }
        }

        // create a new graphic presenting the polyline that is being drawn on the view
        function createLineGraphic(event) {
            const vertices = event.vertices;
            view.graphics.removeAll();

            // a graphic representing the polyline that is being drawn
            const graphic = new Graphic({
                geometry: {
                type: "polyline",
                paths: vertices,
                spatialReference: view.spatialReference
                },
                symbol: {
                type: "simple-line", // autocasts as new SimpleFillSymbol
                color: [4, 90, 141],
                width: 4,
                cap: "round",
                join: "round",
                style: "dash"
                }
            });
            view.graphics.add(graphic);
        }

        function saveLineGraphic(event){
            const vertices = event.vertices;
            view.graphics.removeAll();

            // a graphic representing the polyline that is being drawn
            const graphic = new Graphic({
                geometry: {
                type: "polyline",
                paths: vertices,
                spatialReference: view.spatialReference
                },
                symbol: {
                type: "simple-line", // autocasts as new SimpleFillSymbol
                color: [4, 90, 141],
                width: 4,
                cap: "round",
                join: "round",
                style: "dash"
                }
            });
            layerSavePoint.graphics.add(graphic);
            var param={
              "devType":0,
              "geometryInfo":JSON.stringify(graphic.toJSON()),
              "schemeId":"07b0486b0e014b5ba80685d72dc38351"
            };
            $.ajax({
              url: "http://192.168.43.214:8087/visual/add-project-scheme-dev",
              type: "post",
              data: JSON.stringify(param),
              headers: {
                  'Origin':"*",
                  'content-type': "application/json",
                  // 'token': "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NzQ5MjUzNzAsInVzZXJuYW1lIjoiaHowMDEifQ.P1Rzs0A6klHU6CZqBuItibpya-kfsT5kJ1w7EboJtAo"
              },
              success: function(result) {  //这里就是我出错的地方
                console.log(result.msg);
              },
              error: function(data) {
                console.log("请求出错");
              }
            });
        }
        
        function previewPointGraphic(event){
            var coordinates=event.coordinates;
            view.graphics.removeAll();
            var point = {
                type: "point", // autocasts as /Point
                x: coordinates[0],
                y: coordinates[1],
                spatialReference: view.spatialReference
            };

            var graphic = new Graphic({
                geometry: point,
                symbol: {
                type: "picture-marker", // autocasts as SimpleMarkerSymbol
                url: "https://static.arcgis.com/images/Symbols/Shapes/BlackStarLargeB.png",
                size: "16px",
                }
            });
            view.graphics.add(graphic);            
        }

        function savePointGraphic(event){
            var coordinates=event.coordinates;
            view.graphics.removeAll();
            var point = {
                type: "point", // autocasts as /Point
                x: coordinates[0],
                y: coordinates[1],
                spatialReference: view.spatialReference
            };

            var graphic = new Graphic({
                geometry: point,
                symbol: {
                type: "picture-marker", // autocasts as SimpleMarkerSymbol
                url: "https://static.arcgis.com/images/Symbols/Shapes/BlackStarLargeB.png",
                size: "16px",
                }
            });
            layerSavePoint.graphics.add(graphic);
            var param={
              "devType":0,
              "geometryInfo":JSON.stringify(graphic.toJSON()),
              "schemeId":"07b0486b0e014b5ba80685d72dc38351"
            };
            //console.log(JSON.stringify(graphic.toJSON()));
            $.ajax({
              url: "http://192.168.43.214:8087/visual/add-project-scheme-dev",
              type: "post",
              data: JSON.stringify(param),
              headers: {
                  'Origin':"*",
                  'content-type': "application/json",
                  // 'token': "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NzQ5MjUzNzAsInVzZXJuYW1lIjoiaHowMDEifQ.P1Rzs0A6klHU6CZqBuItibpya-kfsT5kJ1w7EboJtAo"
              },
              success: function(result) {  //这里就是我出错的地方
                console.log(result.msg);
              },
              error: function(data) {
                console.log("请求出错");
              }
            });
            //console.log(JSON.stringify(graphic.toJSON()));
        }
      });
    </script>
  </head>

  <body>
    <div id="viewDiv">
      <div
        id="line-button"
        class="esri-widget esri-widget--button esri-interactive"
        title="Draw polyline"
      >
        <span class="esri-icon-polyline"></span>
      </div>
    </div>
  </body>
</html>

```