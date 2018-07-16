# 概述
我们经常会碰到这样的需求：北京的用户只能查看北京的地图，天津的只能看天津的地图……这里面涉及到了一个地图的访问权限问题，要实现这样的功能如果用服务+过滤的方式比较繁琐，所以本文讲述一种比较简单的实现方式。

# 输入与输出
>输入：地区边界+地图
> 输出：按照地区边界裁剪的地图，并显示地区边界

![实现效果](https://upload-images.jianshu.io/upload_images/6826673-8077a8d76ebe50ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 实现
1、技术关键点
实现此功能中，包含几个关键技术点：
1）地图坐标转换为屏幕坐标；
```js
map.getPixelFromCoordinate(coord);
```
2）canvas绘图中save()、restore()和clip();
```js
var c=document.getElementById("myCanvas");
var ctx=c.getContext("2d");
ctx.save();

ctx.stroke();
ctx.clip();

ctx.restore();
```
2、实现思路
用户登录进来后获取行政区边界，再根据行政区边界进行地图裁剪。

3、地图裁剪代码
```html
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <title>Ol3 wms</title>
    <link rel="stylesheet" type="text/css" href="../../plugin/ol4/ol.css"/>
    <style type="text/css">
        body, #map {
            border: 0px;
            margin: 0px;
            padding: 0px;
            width: 100%;
            height: 100%;
            font-size: 13px;
        }
    </style>
    <script type="text/javascript" src="../../plugin/ol4/ol.js"></script>
    <script type="text/javascript" src="../../plugin/jquery/jquery-3.1.1.min.js"></script>
    <script type="text/javascript">
        var map;
        function init(){
            var vec_c = getTdtLayer("vec_w");
            var cva_c = getTdtLayer("cva_w");
            map = new ol.Map({
                controls: ol.control.defaults({
                    attribution: false
                }),
                target: 'map',
                layers: [vec_c, cva_c],
                view: new ol.View({
                    center: [12967248.127726289, 4891245.816671869],
                    zoom:9,
                    minZoom:0,
                    maxZoom:18
                })
            });

            function getTdtLayer(lyr){
                var url = "http://t{0-7}.tianditu.com/DataServer?T="+lyr+"&X={x}&Y={y}&L={z}";
                var layer = new ol.layer.Tile({
                    source: new ol.source.XYZ({
                        url:url
                    })
                });
                return layer;
            }

            //定义裁剪边界
            var coord = [[[117.315375,40.181212],[117.367916,40.135762],[116.751758,40.002595],[116.754136,39.870341],[116.913383,39.804999],[116.901858,39.680614],[116.788145,39.56255],[116.535646,39.590346],[116.392103,39.491124],[116.4293,39.433841],[116.387072,39.417336],[116.237232,39.476253],[116.172242,39.578854],[115.728745,39.479123],[115.610225,39.588658],[115.508537,39.59137],[115.416399,39.733407],[115.416624,39.776734],[115.550565,39.772964],[115.408433,40.015829],[115.85422,40.144999],[115.922315,40.216777],[115.708758,40.499032],[115.89819,40.585919],[116.03778,40.577909],[116.208725,40.741562],[116.454984,40.739689],[116.297615,40.910402],[116.43816,40.870116],[116.405424,40.94854],[116.537137,40.9661],[116.621495,41.057333],[116.703349,40.853574],[116.93405,40.675155],[117.454502,40.647216],[117.387854,40.533274],[117.166811,40.503979],[117.164198,40.373887],[117.315375,40.181212]]];
            var clipgeom = new ol.geom.Polygon(coord);
            //将经纬度坐标转换为map对应的坐标
            clipgeom = clipgeom.transform("EPSG:4326", map.getView().getProjection());

            map.on('precompose', clip);

            function clip(evt) {
                var canvas=evt.context;
                canvas.save();
                var coords = clipgeom.getCoordinates();
                canvas.beginPath();
                createClip(coords[0], canvas);
                canvas.clip();
            }

            function createClip(coords, canvas) {
                for (var i = 0, cout = coords.length; i < cout; i++) {
                    //获取屏幕坐标
                    var screenCoord = map.getPixelFromCoordinate(coords[i]);
                    var x = screenCoord[0],
                        y = screenCoord[1];
                    if (i === 0) {
                        canvas.moveTo(x, y);
                    } else {
                        canvas.lineTo(x, y);
                    }
                }
                canvas.closePath();
                //设置边框
                canvas.strokeStyle = "red";
                canvas.lineWidth = 2;
                canvas.stroke();
            }
            map.on('postcompose', function(event) {
                var ctx = event.context;
                ctx.restore();
            });
        }
    </script>
</head>
<body onLoad="init()">
<div id="map">
</div>
</body>
</html>
```