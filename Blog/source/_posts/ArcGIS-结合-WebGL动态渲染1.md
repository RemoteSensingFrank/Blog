---
title: ArcGIS 结合 WebGL动态渲染1
date: 2020-10-30 11:51:33
tags: ArcGIS，前端开发
categories: 学习
---
&nbsp;&nbsp;&nbsp;&nbsp;目前GIS应用过程中，对于渲染效果的要求越来越高最好能够采用一些动画来实现渲染效果的优化，而在ArcGIS js API中直接通过FeatureLayer只能够渲染一些简单的图标，难以完成动画效果的展示，为了能够展示比较好的渲染效果，我对官网DEMO进行了仔细研究，发现实际上如果能够与WebGL进行深入结合并进行定制化的开发，采用ArcGIS js API也能够有很好的效果，[具体官网链接参考](https://developers.arcgis.com/javascript/latest/sample-code/?search=WebGL)，本次我们介绍在MapView下结合WebGL的渲染方式，主要效果展示为：  
1. ArcGIS 点强调动态效果展示：
![ArcGIS WebGL 点强调展示](https://blogimage-1251632003.cos.ap-guangzhou.myqcloud.com/arcgis%E7%82%B9%E5%BC%BA%E8%B0%83.gif)
2. ArcGIS WebGL 线流动态效果展示：
![ArcGIS WebGL 线流展示](https://blogimage-1251632003.cos.ap-guangzhou.myqcloud.com/arcgis%E7%BA%BF%E6%B5%81%E5%9B%BE.gif)  

上面两幅图为展示的动态效果，下面我们针对动态展示效果的代码进行分析：
```javascript
require([
"esri/Map",

"esri/core/watchUtils",
"esri/core/promiseUtils",

"esri/geometry/support/webMercatorUtils",

"esri/layers/GraphicsLayer",

"esri/views/MapView",

"esri/views/2d/layers/BaseLayerViewGL2D"
], function (
Map,
watchUtils,
promiseUtils,
webMercatorUtils,
GraphicsLayer,
MapView,
BaseLayerViewGL2D
) {
// Subclass the custom layer view from BaseLayerViewGL2D.
var CustomLayerView2D = BaseLayerViewGL2D.createSubclass({
    // Locations of the two vertex attributes that we use. They
    // will be bound to the shader program before linking.
    aPosition: 0,
    aOffset: 1,
    aColor:2,
    constructor: function () {
    // Geometrical transformations that must be recomputed
    // from scratch at every frame.
    this.transform = mat3.create();
    this.translationToCenter = vec2.create();
    this.screenTranslation = vec2.create();

    // Geometrical transformations whose only a few elements
    // must be updated per frame. Those elements are marked
    // with NaN.
    this.display = mat3.fromValues(NaN, 0, 0, 0, NaN, 0, -1, 1, 1);
    this.screenScaling = vec3.fromValues(NaN, NaN, 1);

    // Whether the vertex and index buffers need to be updated
    // due to a change in the layer data.
    this.needsUpdate = false;

    // We listen for changes to the graphics collection of the layer
    // and trigger the generation of new frames. A frame rendered while
    // `needsUpdate` is true may cause an update of the vertex and
    // index buffers.
    var requestUpdate = function () {
        this.needsUpdate = true;
        this.requestRender();
    }.bind(this);

    this.watcher = watchUtils.on(
        this,
        "layer.graphics",
        "change",
        requestUpdate,
        requestUpdate,
        requestUpdate
    );
    },

    // Called once a custom layer is added to the map.layers collection and this layer view is instantiated.
    attach: function () {
    var gl = this.context;

    // Define and compile shaders.
    var vertexSource =
        "precision highp float;" +
        "uniform mat3 u_transform;" +
        "uniform mat3 u_display;" +
        "attribute vec2 a_position;" +
        "attribute vec2 a_offset;" +
        "attribute vec3 a_color;" +
        "varying vec2 v_offset;" +
        "varying vec3 v_color;" +
        "const float SIZE = 70.0;" +
        "void main() {" +
        "    gl_Position.xy = (u_display * (u_transform * vec3(a_position, 1.0) + vec3(a_offset * SIZE, 0.0))).xy;" +
        "    gl_Position.zw = vec2(0.0, 1.0);" +
        "    v_offset = a_offset;" +
        "    v_color = a_color;" +
        "}";

    var fragmentSource =
        "precision highp float;" +
        "uniform float u_current_time;" +
        "varying vec2 v_offset;" +
        "varying vec3 v_color;" +
        "const float PI = 3.14159;" +
        "const float N_RINGS = 3.0;" +
        "const vec3 COLOR = vec3(0.23, 0.43, 0.70);" +
        "const float FREQ = 1.0;" +
        "void main() {" +
        "    float l = length(v_offset);" +
        "    float intensity = clamp(cos(l * PI), 0.0, 1.0) * clamp(cos(2.0 * PI * (l * 2.0 * N_RINGS - FREQ * u_current_time)), 0.0, 1.0);" +
        "    gl_FragColor = vec4(v_color * intensity, intensity);" +
        "}";

    var vertexShader = gl.createShader(gl.VERTEX_SHADER);
    gl.shaderSource(vertexShader, vertexSource);
    gl.compileShader(vertexShader);
    var fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
    gl.shaderSource(fragmentShader, fragmentSource);
    gl.compileShader(fragmentShader);

    // Create the shader program.
    this.program = gl.createProgram();
    gl.attachShader(this.program, vertexShader);
    gl.attachShader(this.program, fragmentShader);

    // Bind attributes.
    gl.bindAttribLocation(this.program, this.aPosition, "a_position");
    gl.bindAttribLocation(this.program, this.aOffset, "a_offset");
    gl.bindAttribLocation(this.program, this.aColor, "a_color");

    // Link.
    gl.linkProgram(this.program);

    // Shader objects are not needed anymore.
    gl.deleteShader(vertexShader);
    gl.deleteShader(fragmentShader);

    // Retrieve uniform locations once and for all.
    this.uTransform = gl.getUniformLocation(
        this.program,
        "u_transform"
    );
    this.uDisplay = gl.getUniformLocation(this.program, "u_display");
    this.uCurrentTime = gl.getUniformLocation(
        this.program,
        "u_current_time"
    );

    // Create the vertex and index buffer. They are initially empty. We need to track the
    // size of the index buffer because we use indexed drawing.
    this.vertexBuffer = gl.createBuffer();
    this.indexBuffer = gl.createBuffer();

    // Number of indices in the index buffer.
    this.indexBufferSize = 0;

    // When certain conditions occur, we update the buffers and re-compute and re-encode
    // all the attributes. When buffer update occurs, we also take note of the current center
    // of the view state, and we reset a vector called `translationToCenter` to [0, 0], meaning that the
    // current center is the same as it was when the attributes were recomputed.
    this.centerAtLastUpdate = vec2.fromValues(
        this.view.state.center[0],
        this.view.state.center[1]
    );
    },

    // Called once a custom layer is removed from the map.layers collection and this layer view is destroyed.
    detach: function () {
    // Stop watching the `layer.graphics` collection.
    this.watcher.remove();

    var gl = this.context;

    // Delete buffers and programs.
    gl.deleteBuffer(this.vertexBuffer);
    gl.deleteBuffer(this.indexBuffer);
    gl.deleteProgram(this.program);
    },

    // Called every time a frame is rendered.
    render: function (renderParameters) {
    var gl = renderParameters.context;
    var state = renderParameters.state;

    // Update vertex positions. This may trigger an update of
    // the vertex coordinates contained in the vertex buffer.
    // There are three kinds of updates:
    //  - Modification of the layer.graphics collection ==> Buffer update
    //  - The view state becomes non-stationary ==> Only view update, no buffer update
    //  - The view state becomes stationary ==> Buffer update
    this.updatePositions(renderParameters);

    // If there is nothing to render we return.
    if (this.indexBufferSize === 0) {
        return;
    }

    // Update view `transform` matrix; it converts from map units to pixels.
    mat3.identity(this.transform);
    this.screenTranslation[0] = (state.pixelRatio * state.size[0]) / 2;
    this.screenTranslation[1] = (state.pixelRatio * state.size[1]) / 2;
    mat3.translate(
        this.transform,
        this.transform,
        this.screenTranslation
    );
    mat3.rotate(
        this.transform,
        this.transform,
        (Math.PI * state.rotation) / 180
    );
    this.screenScaling[0] = state.pixelRatio / state.resolution;
    this.screenScaling[1] = -state.pixelRatio / state.resolution;
    mat3.scale(this.transform, this.transform, this.screenScaling);
    mat3.translate(
        this.transform,
        this.transform,
        this.translationToCenter
    );

    // Update view `display` matrix; it converts from pixels to normalized device coordinates.
    this.display[0] = 2 / (state.pixelRatio * state.size[0]);
    this.display[4] = -2 / (state.pixelRatio * state.size[1]);

    // Draw.
    gl.useProgram(this.program);
    gl.uniformMatrix3fv(this.uTransform, false, this.transform);
    gl.uniformMatrix3fv(this.uDisplay, false, this.display);
    gl.uniform1f(this.uCurrentTime, performance.now() / 1000.0);
    gl.bindBuffer(gl.ARRAY_BUFFER, this.vertexBuffer);
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, this.indexBuffer);
    gl.enableVertexAttribArray(this.aPosition);
    gl.enableVertexAttribArray(this.aOffset);
    gl.enableVertexAttribArray(this.aColor);

    gl.vertexAttribPointer(this.aPosition, 2, gl.FLOAT, false, 28, 0);
    gl.vertexAttribPointer(this.aOffset, 2, gl.FLOAT, false, 28, 8);
    gl.vertexAttribPointer(this.aColor,3,gl.FLOAT,false,28,16);

    gl.enable(gl.BLEND);
    gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
    gl.drawElements(
        gl.TRIANGLES,
        this.indexBufferSize,
        gl.UNSIGNED_SHORT,
        0
    );

    // Request new render because markers are animated.
    this.requestRender();
    },

    // Called by the map view or the popup view when hit testing is required.
    hitTest: function (x, y) {
    // The map view.
    var view = this.view;

    if (this.layer.graphics.length === 0) {
        // Nothing to do.
        return promiseUtils.resolve(null);
    }

    // Compute screen distance between each graphic and the test point.
    var distances = this.layer.graphics.map(function (graphic) {
        var graphicPoint = view.toScreen(graphic.geometry);
        return Math.sqrt(
        (graphicPoint.x - x) * (graphicPoint.x - x) +
            (graphicPoint.y - y) * (graphicPoint.y - y)
        );
    });

    // Find the minimum distance.
    var minIndex = 0;

    distances.forEach(function (distance, i) {
        if (distance < distances.getItemAt(minIndex)) {
        minIndex = i;
        }
    });

    var minDistance = distances.getItemAt(minIndex);

    // If the minimum distance is more than 35 pixel then nothing was hit.
    if (minDistance > 35) {
        return promiseUtils.resolve(null);
    }

    // Otherwise it is a hit; We set the layer as the source layer for the graphic
    // (required for the popup view to work) and we return a resolving promise to
    // the graphic.
    var graphic = this.layer.graphics.getItemAt(minIndex);
    graphic.sourceLayer = this.layer;
    return promiseUtils.resolve(graphic);
    },

    // Called internally from render().
    updatePositions: function (renderParameters) {
    var gl = renderParameters.context;
    var stationary = renderParameters.stationary;
    var state = renderParameters.state;

    // If we are not stationary we simply update the `translationToCenter` vector.
    if (!stationary) {
        vec2.sub(
        this.translationToCenter,
        this.centerAtLastUpdate,
        state.center
        );
        this.requestRender();
        return;
    }

    // If we are stationary, the `layer.graphics` collection has not changed, and
    // we are centered on the `centerAtLastUpdate`, we do nothing.
    if (
        !this.needsUpdate &&
        this.translationToCenter[0] === 0 &&
        this.translationToCenter[1] === 0
    ) {
        return;
    }

    // Otherwise, we record the new encoded center, which imply a reset of the `translationToCenter` vector,
    // we record the update time, and we proceed to update the buffers.
    this.centerAtLastUpdate.set(state.center);
    this.translationToCenter[0] = 0;
    this.translationToCenter[1] = 0;
    this.needsUpdate = false;

    var graphics = this.layer.graphics;

    // Generate vertex data.
    // gl.bindBuffer(gl.ARRAY_BUFFER, this.vertexBuffer);

    var vertexData = new ArrayBuffer(7*4 * graphics.length*4);
    var floatData = new Float32Array(vertexData);
    var colorData = new Float32Array(vertexData);

    var i = 0;
    graphics.forEach(
        function (graphic) {
        var point = graphic.geometry;

        // The (x, y) position is relative to the encoded center.
        var x = point.x - this.centerAtLastUpdate[0];
        var y = point.y - this.centerAtLastUpdate[1];

        floatData[i * 28 + 0] = x;
        floatData[i * 28 + 1] = y;
        floatData[i * 28 + 2] = -0.5;
        floatData[i * 28 + 3] = -0.5;
        colorData[i * 28 + 4] = 0.5;
        colorData[i * 28 + 5] = 0.43;
        colorData[i * 28 + 6] = 0.70;

        floatData[i * 28 + 7] = x;
        floatData[i * 28 + 8] = y;
        floatData[i * 28 + 9] = 0.5;
        floatData[i * 28 + 10] = -0.5;
        colorData[i * 28 + 11] = 1;
        colorData[i * 28 + 12] = 0.43;
        colorData[i * 28 + 13] = 0.70;

        floatData[i * 28 + 14] = x;
        floatData[i * 28 + 15] = y;
        floatData[i * 28 + 16] = -0.5;
        floatData[i * 28 + 17] = 0.5;
        colorData[i * 28 + 18] = 1;
        colorData[i * 28 + 19] = 0.43;
        colorData[i * 28 + 20] = 0.70;

        floatData[i * 28 + 21] = x;
        floatData[i * 28 + 22] = y;
        floatData[i * 28 + 23] = 0.5;
        floatData[i * 28 + 24] = 0.5;
        colorData[i * 28 + 25] = 1;
        colorData[i * 28 + 26] = 0.43;
        colorData[i * 28 + 27] = 0.70;

        ++i;
        }.bind(this)
    );

    gl.bindBuffer(gl.ARRAY_BUFFER, this.vertexBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, vertexData, gl.STATIC_DRAW);

    // Generates index data.
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, this.indexBuffer);

    var indexData = new Uint16Array(6 * graphics.length);
    for (var i = 0; i < graphics.length; ++i) {
        indexData[i * 6 + 0] = i * 4 + 0;
        indexData[i * 6 + 1] = i * 4 + 1;
        indexData[i * 6 + 2] = i * 4 + 2;
        indexData[i * 6 + 3] = i * 4 + 1;
        indexData[i * 6 + 4] = i * 4 + 3;
        indexData[i * 6 + 5] = i * 4 + 2;
    }

    gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, indexData, gl.STATIC_DRAW);

    // Record number of indices.
    this.indexBufferSize = indexData.length;
    }
});

// Subclass the custom layer view from GraphicsLayer.
var CustomLayer = GraphicsLayer.createSubclass({
    createLayerView: function (view) {
    // We only support MapView, so we only need to return a
    // custom layer view for the `2d` case.
    if (view.type === "2d") {
        return new CustomLayerView2D({
        view: view,
        layer: this
        });
    }
    }
});

// Create an instance of the custom layer with 4 initial graphics.
var layer = new CustomLayer({
    popupTemplate: {
    title: "{NAME}",
    content: "Population: {POPULATION}."
    },
    graphics: [
    {
        geometry: webMercatorUtils.geographicToWebMercator({
        // Los Angeles
        x: -118.2437,
        y: 34.0522,
        type: "point",
        spatialReference: {
            wkid: 4326
        }
        }),
        attributes: {
        NAME: "Los Angeles",
        POPULATION: 3792621
        }
    },
    {
        geometry: webMercatorUtils.geographicToWebMercator({
        // Dallas
        x: -96.797,
        y: 32.7767,
        type: "point",
        spatialReference: {
            wkid: 4326
        }
        }),
        attributes: {
        NAME: "Dallas",
        POPULATION: 1197816
        }
    },
    {
        geometry: webMercatorUtils.geographicToWebMercator({
        // Denver
        x: -104.9903,
        y: 39.7392,
        type: "point",
        spatialReference: {
            wkid: 4326
        }
        }),
        attributes: {
        NAME: "Denver",
        POPULATION: 600158
        }
    },
    {
        geometry: webMercatorUtils.geographicToWebMercator({
        // New York
        x: -74.006,
        y: 40.7128,
        type: "point",
        spatialReference: {
            wkid: 4326
        }
        }),
        attributes: {
        NAME: "New York",
        POPULATION: 8175133
        }
    }
    ]
});

// Create the map and the view.
var map = new Map({
    basemap: "streets-night-vector",
    layers: [layer]
});

var view = new MapView({
    container: "viewDiv",
    map: map,
    center: [-100, 40],
    zoom: 3
});

var lastFeatureId = 0;

// Add new graphics on double click.
view.on(
    "double-click",
    function (event) {
    event.stopPropagation();

    ++lastFeatureId;

    layer.graphics.add({
        geometry: event.mapPoint,
        attributes: {
        NAME: "Feature #" + lastFeatureId,
        POPULATION: 100000
        }
    });
    }.bind(this)
);
});
```
整个代码的主体功能在于构造一个自定义的BaseLayerViewGL2D结构，将WebGL渲染嵌入我们自定义的图层中，然后重构对应的渲染方法进行渲染，我们先查看流程结构功能代码：
```javascript
// Subclass the custom layer view from GraphicsLayer.
var CustomLayer = GraphicsLayer.createSubclass({
    createLayerView: function (view) {
    // We only support MapView, so we only need to return a
    // custom layer view for the `2d` case.
    if (view.type === "2d") {
        return new CustomLayerView2D({
        view: view,
        layer: this
        });
    }
    }
});

// Create an instance of the custom layer with 4 initial graphics.
var layer = new CustomLayer({
    popupTemplate: {
    title: "{NAME}",
    content: "Population: {POPULATION}."
    },
    graphics: [
    {
        geometry: webMercatorUtils.geographicToWebMercator({
        // Los Angeles
        x: -118.2437,
        y: 34.0522,
        type: "point",
        spatialReference: {
            wkid: 4326
        }
        }),
        attributes: {
        NAME: "Los Angeles",
        POPULATION: 3792621
        }
    },
    {...}
    ]
});

// Create the map and the view.
var map = new Map({
    basemap: "streets-night-vector",
    layers: [layer]
});

var view = new MapView({
    container: "viewDiv",
    map: map,
    center: [-100, 40],
    zoom: 3
});
```
实际上添加图层的过程相对比较简单，首先根据前面定义的自定义layer，CustomLayerView2D创建一个layer类，然后往其中添加graphic数据，最后将图层添加到map中，最后在view中进行展示，所以整个流程可以描述为[自定义WebGL绘制图层]->[新建图层]->[初始化图层数据（往图层中添加数据）]->[将图层添加到map中]其他的步骤都跟添加一个普通的图层没有区别，其中最核心的部分在于“自定义WebGL绘制图层”，我们把定义绘制图层的部分截取出来并进行详细的注释描述
```javascript
var CustomLayerView2D = BaseLayerViewGL2D.createSubclass({
    // Locations of the two vertex attributes that we use. They
    // will be bound to the shader program before linking.
    
    //在这里定义三个位置，实际上这三个变量只是占位索引而已，所以编码是从0-N
    aPosition: 0,
    aOffset: 1,
    aColor:2,

    //重构的构造函数
    constructor: function () {

    // Geometrical transformations that must be recomputed
    // from scratch at every frame.
    // 这里有两个转换，WebGL的坐标是屏幕坐标而实际上点的坐标为经纬度或web mector投影坐标，所以转换的过程为
    // 如果为经纬度则首先转换为web mector，然后去中心化，然后计算为屏幕坐标
    this.transform = mat3.create();
    this.translationToCenter = vec2.create();
    this.screenTranslation = vec2.create();

    // Geometrical transformations whose only a few elements
    // must be updated per frame. Those elements are marked
    // with NaN.
    this.display = mat3.fromValues(NaN, 0, 0, 0, NaN, 0, -1, 1, 1);
    this.screenScaling = vec3.fromValues(NaN, NaN, 1);

    // Whether the vertex and index buffers need to be updated
    // due to a change in the layer data.
    this.needsUpdate = false;

    // We listen for changes to the graphics collection of the layer
    // and trigger the generation of new frames. A frame rendered while
    // `needsUpdate` is true may cause an update of the vertex and
    // index buffers.
    var requestUpdate = function () {
        this.needsUpdate = true;
        this.requestRender();
    }.bind(this);

    this.watcher = watchUtils.on(
        this,
        "layer.graphics",
        "change",
        requestUpdate,
        requestUpdate,
        requestUpdate
    );
    },

    // Called once a custom layer is added to the map.layers collection and this layer view is instantiated.
    attach: function () {
    var gl = this.context;

    // Define and compile shaders.
    // 实际上这个部分就是初始化gl并定义渲染方法的部分，
    // 在此部分定义传入变量，包括a_position，a_offset，a_color
    // 然后根据传入遍历对着色器进行渲染最后得到渲染后的结果
    var vertexSource =
        "precision highp float;" +
        "uniform mat3 u_transform;" +
        "uniform mat3 u_display;" +
        "attribute vec2 a_position;" +
        "attribute vec2 a_offset;" +
        "attribute vec3 a_color;" +
        "varying vec2 v_offset;" +
        "varying vec3 v_color;" +
        "const float SIZE = 70.0;" +
        "void main() {" +
        "    gl_Position.xy = (u_display * (u_transform * vec3(a_position, 1.0) + vec3(a_offset * SIZE, 0.0))).xy;" +
        "    gl_Position.zw = vec2(0.0, 1.0);" +
        "    v_offset = a_offset;" +
        "    v_color = a_color;" +
        "}";

    var fragmentSource =
        "precision highp float;" +
        "uniform float u_current_time;" +
        "varying vec2 v_offset;" +
        "varying vec3 v_color;" +
        "const float PI = 3.14159;" +
        "const float N_RINGS = 3.0;" +
        "const vec3 COLOR = vec3(0.23, 0.43, 0.70);" +
        "const float FREQ = 1.0;" +
        "void main() {" +
        "    float l = length(v_offset);" +
        "    float intensity = clamp(cos(l * PI), 0.0, 1.0) * clamp(cos(2.0 * PI * (l * 2.0 * N_RINGS - FREQ * u_current_time)), 0.0, 1.0);" +
        "    gl_FragColor = vec4(v_color * intensity, intensity);" +
        "}";

    var vertexShader = gl.createShader(gl.VERTEX_SHADER);
    gl.shaderSource(vertexShader, vertexSource);
    gl.compileShader(vertexShader);
    var fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
    gl.shaderSource(fragmentShader, fragmentSource);
    gl.compileShader(fragmentShader);

    // Create the shader program.
    this.program = gl.createProgram();
    gl.attachShader(this.program, vertexShader);
    gl.attachShader(this.program, fragmentShader);

    // Bind attributes.
    // 此部分要注意，绑定渲染器的顺序一定要按照前面定义的遍历的顺序来0-N的顺序
    gl.bindAttribLocation(this.program, this.aPosition, "a_position");
    gl.bindAttribLocation(this.program, this.aOffset, "a_offset");
    gl.bindAttribLocation(this.program, this.aColor, "a_color");

    // Link.
    gl.linkProgram(this.program);

    // Shader objects are not needed anymore.
    gl.deleteShader(vertexShader);
    gl.deleteShader(fragmentShader);

    // Retrieve uniform locations once and for all.
    // 如果字节映射出问题这里经常会报一个警告，
    // 所以这边出现警告的时候需要结合前后代码进行判断
    this.uTransform = gl.getUniformLocation(
        this.program,
        "u_transform"
    );
    this.uDisplay = gl.getUniformLocation(this.program, "u_display");
    this.uCurrentTime = gl.getUniformLocation(
        this.program,
        "u_current_time"
    );

    // Create the vertex and index buffer. They are initially empty. We need to track the
    // size of the index buffer because we use indexed drawing.
    this.vertexBuffer = gl.createBuffer();
    this.indexBuffer = gl.createBuffer();

    // Number of indices in the index buffer.
    this.indexBufferSize = 0;

    // When certain conditions occur, we update the buffers and re-compute and re-encode
    // all the attributes. When buffer update occurs, we also take note of the current center
    // of the view state, and we reset a vector called `translationToCenter` to [0, 0], meaning that the
    // current center is the same as it was when the attributes were recomputed.
    this.centerAtLastUpdate = vec2.fromValues(
        this.view.state.center[0],
        this.view.state.center[1]
    );
    },

    // Called once a custom layer is removed from the map.layers collection and this layer view is destroyed.
    detach: function () {
    // Stop watching the `layer.graphics` collection.
    this.watcher.remove();

    var gl = this.context;

    // Delete buffers and programs.
    gl.deleteBuffer(this.vertexBuffer);
    gl.deleteBuffer(this.indexBuffer);
    gl.deleteProgram(this.program);
    },

    // Called every time a frame is rendered.
    render: function (renderParameters) {

    //此函数中主要是渲染和绘制的过程，具体的渲染方式以及转换矩阵的操作可以去参考    
    var gl = renderParameters.context;
    var state = renderParameters.state;

    // Update vertex positions. This may trigger an update of
    // the vertex coordinates contained in the vertex buffer.
    // There are three kinds of updates:
    //  - Modification of the layer.graphics collection ==> Buffer update
    //  - The view state becomes non-stationary ==> Only view update, no buffer update
    //  - The view state becomes stationary ==> Buffer update
    this.updatePositions(renderParameters);

    // If there is nothing to render we return.
    if (this.indexBufferSize === 0) {
        return;
    }

    // Update view `transform` matrix; it converts from map units to pixels.
    mat3.identity(this.transform);
    this.screenTranslation[0] = (state.pixelRatio * state.size[0]) / 2;
    this.screenTranslation[1] = (state.pixelRatio * state.size[1]) / 2;
    mat3.translate(
        this.transform,
        this.transform,
        this.screenTranslation
    );
    mat3.rotate(
        this.transform,
        this.transform,
        (Math.PI * state.rotation) / 180
    );
    this.screenScaling[0] = state.pixelRatio / state.resolution;
    this.screenScaling[1] = -state.pixelRatio / state.resolution;
    mat3.scale(this.transform, this.transform, this.screenScaling);
    mat3.translate(
        this.transform,
        this.transform,
        this.translationToCenter
    );

    // Update view `display` matrix; it converts from pixels to normalized device coordinates.
    this.display[0] = 2 / (state.pixelRatio * state.size[0]);
    this.display[4] = -2 / (state.pixelRatio * state.size[1]);

    // Draw.
    gl.useProgram(this.program);
    gl.uniformMatrix3fv(this.uTransform, false, this.transform);
    gl.uniformMatrix3fv(this.uDisplay, false, this.display);
    gl.uniform1f(this.uCurrentTime, performance.now() / 1000.0);
    gl.bindBuffer(gl.ARRAY_BUFFER, this.vertexBuffer);
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, this.indexBuffer);
    
    // 判定变量
    gl.enableVertexAttribArray(this.aPosition);
    gl.enableVertexAttribArray(this.aOffset);
    gl.enableVertexAttribArray(this.aColor);

    // 字节映射
    gl.vertexAttribPointer(this.aPosition, 2, gl.FLOAT, false, 28, 0);
    gl.vertexAttribPointer(this.aOffset, 2, gl.FLOAT, false, 28, 8);
    gl.vertexAttribPointer(this.aColor,3,gl.FLOAT,false,28,16);

    gl.enable(gl.BLEND);
    gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
    gl.drawElements(
        gl.TRIANGLES,
        this.indexBufferSize,
        gl.UNSIGNED_SHORT,
        0
    );

    // Request new render because markers are animated.
    this.requestRender();
    },

    // Called by the map view or the popup view when hit testing is required.
    hitTest: function (x, y) {
    // The map view.
    var view = this.view;

    if (this.layer.graphics.length === 0) {
        // Nothing to do.
        return promiseUtils.resolve(null);
    }

    // Compute screen distance between each graphic and the test point.
    var distances = this.layer.graphics.map(function (graphic) {
        var graphicPoint = view.toScreen(graphic.geometry);
        return Math.sqrt(
        (graphicPoint.x - x) * (graphicPoint.x - x) +
            (graphicPoint.y - y) * (graphicPoint.y - y)
        );
    });

    // Find the minimum distance.
    var minIndex = 0;

    distances.forEach(function (distance, i) {
        if (distance < distances.getItemAt(minIndex)) {
        minIndex = i;
        }
    });

    var minDistance = distances.getItemAt(minIndex);

    // If the minimum distance is more than 35 pixel then nothing was hit.
    if (minDistance > 35) {
        return promiseUtils.resolve(null);
    }

    // Otherwise it is a hit; We set the layer as the source layer for the graphic
    // (required for the popup view to work) and we return a resolving promise to
    // the graphic.
    // 点击事件，返回graphic
    var graphic = this.layer.graphics.getItemAt(minIndex);
    graphic.sourceLayer = this.layer;
    return promiseUtils.resolve(graphic);
    },

    // Called internally from render().
    updatePositions: function (renderParameters) {
    var gl = renderParameters.context;
    var stationary = renderParameters.stationary;
    var state = renderParameters.state;

    // If we are not stationary we simply update the `translationToCenter` vector.
    if (!stationary) {
        vec2.sub(
        this.translationToCenter,
        this.centerAtLastUpdate,
        state.center
        );
        this.requestRender();
        return;
    }

    // If we are stationary, the `layer.graphics` collection has not changed, and
    // we are centered on the `centerAtLastUpdate`, we do nothing.
    if (
        !this.needsUpdate &&
        this.translationToCenter[0] === 0 &&
        this.translationToCenter[1] === 0
    ) {
        return;
    }

    // Otherwise, we record the new encoded center, which imply a reset of the `translationToCenter` vector,
    // we record the update time, and we proceed to update the buffers.
    this.centerAtLastUpdate.set(state.center);
    this.translationToCenter[0] = 0;
    this.translationToCenter[1] = 0;
    this.needsUpdate = false;

    var graphics = this.layer.graphics;

    // Generate vertex data.
    // gl.bindBuffer(gl.ARRAY_BUFFER, this.vertexBuffer);
    // 一共有多少字节，实际上每一个点有两个坐标x，y，然后两个方向变量，再加3个float的RGB颜色变量，每次绘制四个点
    var vertexData = new ArrayBuffer(7*4 * graphics.length*4);
    // 映射坐标和方向变量
    var floatData = new Float32Array(vertexData);
    // 颜色变量
    var colorData = new Float32Array(vertexData);

    var i = 0;
    graphics.forEach(
        function (graphic) {
        var point = graphic.geometry;

        // The (x, y) position is relative to the encoded center.
        var x = point.x - this.centerAtLastUpdate[0];
        var y = point.y - this.centerAtLastUpdate[1];

        floatData[i * 28 + 0] = x;
        floatData[i * 28 + 1] = y;
        floatData[i * 28 + 2] = -0.5;
        floatData[i * 28 + 3] = -0.5;
        colorData[i * 28 + 4] = 0.5;
        colorData[i * 28 + 5] = 0.43;
        colorData[i * 28 + 6] = 0.70;

        floatData[i * 28 + 7] = x;
        floatData[i * 28 + 8] = y;
        floatData[i * 28 + 9] = 0.5;
        floatData[i * 28 + 10] = -0.5;
        colorData[i * 28 + 11] = 1;
        colorData[i * 28 + 12] = 0.43;
        colorData[i * 28 + 13] = 0.70;

        floatData[i * 28 + 14] = x;
        floatData[i * 28 + 15] = y;
        floatData[i * 28 + 16] = -0.5;
        floatData[i * 28 + 17] = 0.5;
        colorData[i * 28 + 18] = 1;
        colorData[i * 28 + 19] = 0.43;
        colorData[i * 28 + 20] = 0.70;

        floatData[i * 28 + 21] = x;
        floatData[i * 28 + 22] = y;
        floatData[i * 28 + 23] = 0.5;
        floatData[i * 28 + 24] = 0.5;
        colorData[i * 28 + 25] = 1;
        colorData[i * 28 + 26] = 0.43;
        colorData[i * 28 + 27] = 0.70;

        ++i;
        }.bind(this)
    );

    // 绑定数据
    gl.bindBuffer(gl.ARRAY_BUFFER, this.vertexBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, vertexData, gl.STATIC_DRAW);

    // Generates index data.
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, this.indexBuffer);

    // 绘制的映射
    var indexData = new Uint16Array(6 * graphics.length);
    for (var i = 0; i < graphics.length; ++i) {
        indexData[i * 6 + 0] = i * 4 + 0;
        indexData[i * 6 + 1] = i * 4 + 1;
        indexData[i * 6 + 2] = i * 4 + 2;
        indexData[i * 6 + 3] = i * 4 + 1;
        indexData[i * 6 + 4] = i * 4 + 3;
        indexData[i * 6 + 5] = i * 4 + 2;
    }

    gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, indexData, gl.STATIC_DRAW);

    // Record number of indices.
    this.indexBufferSize = indexData.length;
    }
});
```