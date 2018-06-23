# 概述
本文分享一个结合turf.js实现前端等值线的生成，并对结果做了圆滑处理，并在OL4中进行展示。

# 效果
![圆滑前](https://upload-images.jianshu.io/upload_images/6826673-09b894989886ad41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![圆滑后](https://upload-images.jianshu.io/upload_images/6826673-860162606a75efa8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 实现
实现比较简单，源代码如下：
```html
<!DOCTYPE html>
<html>
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<title>Measure</title>
	<link rel="stylesheet" href="../../plugin/ol4/ol.css" type="text/css">
	<script src="../../plugin/ol4/ol.js"></script>
	<script src="../../plugin/turf/turf.js"></script>
	<script type="text/javascript" src="../../plugin/jquery/jquery-3.1.1.min.js"></script>
	<style>
		html, body, #map{
			margin: 0;
			padding: 0;
			overflow: hidden;
			width: 100%;
			height: 100%;
		}
	</style>
</head>
<body>
<div id="map" class="map"></div>
<script>
    function styleFunction(feature) {
        var tem = feature.get("temperature");
        var colors = ["#5a9fdd", "#f7eb0c", "#fc9e10", "#f81e0d", "#aa2ab3"];
        var color = colors[parseInt(tem/2)];
        return new ol.style.Style({
            fill: new ol.style.Fill({
                color: color
            }),
            stroke: new ol.style.Stroke({
                color: color,
                width: 4
            }),
            image: new ol.style.Circle({
                radius: 5,
                fill: new ol.style.Fill({
                    color: color
                }),
                stroke: new ol.style.Stroke({
                    color: '#fff',
                    width: 1
                })
            })
        });
    }

    // create a grid of points with random z-values in their properties
    var extent = [0, 30, 20, 50];
    var cellWidth = 100;
    var pointGrid = turf.pointGrid(extent, cellWidth, {units: 'miles'});
    for (var i = 0; i < pointGrid.features.length; i++) {
        pointGrid.features[i].properties.temperature = Math.random() * 10;
    }
    var breaks = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    var lines = turf.isolines(pointGrid, breaks, {zProperty: 'temperature'});

//    var _lFeatures = lines.features;
//    for(var i=0;i<_lFeatures.length;i++){
//        var _coords = _lFeatures[i].geometry.coordinates;
//        var _lCoords = [];
//        for(var j=0;j<_coords.length;j++){
//            var _coord = _coords[j];
//            var line = turf.lineString(_coord);
//            var curved = turf.bezierSpline(line);
//            _lCoords.push(curved.geometry.coordinates);
//        }
//        _lFeatures[i].geometry.coordinates = _lCoords;
//    }

    var featureslines = (new ol.format.GeoJSON()).readFeatures(lines);
    var features = (new ol.format.GeoJSON()).readFeatures(pointGrid);
    features = features.concat(featureslines);
    for(var i=0;i<features.length;i++){
        features[i].getGeometry().transform("EPSG:4326","EPSG:3857");
    }
    var source = new ol.source.Vector({
        features:features
    });
    var vector = new ol.layer.Vector({
        source: source,
        style: styleFunction
    });
    var map = new ol.Map({
        layers: [getTdtLayer("img_w"), vector],
        target: 'map',
        view: new ol.View({
            center: [12577713.642017495, 2971206.770222437],
            zoom: 3
        })
    });
    function getTdtLayer(lyr){
        var url = "http://t0.tianditu.com/DataServer?T="+lyr+"&X={x}&Y={y}&L={z}";
        var layer = new ol.layer.Tile({
            source: new ol.source.XYZ({
                url:url
            })
        });
        return layer;
    }
</script>
</body>
</html>
```
**说明：**
1.注释掉的代码为做圆滑处理的代码，拷贝请取消注释。