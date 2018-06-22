# 概述
虽然Openlayers4会有自带的聚类效果，但是有些时候是不能满足我们的业务场景的，本文结合一些业务场景，讲讲地图中的聚类展示。

# 需求
在级别比较小的时候聚类展示数据，当级别大于一定的级别的时候讲地图可视域内的所有点不做聚类全部展示出来。

# 效果
![小级别](https://upload-images.jianshu.io/upload_images/6826673-58c48a1e0b7d3d7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![放大地图](https://upload-images.jianshu.io/upload_images/6826673-cb3fa4ea491f68df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![固定级别全部展示](https://upload-images.jianshu.io/upload_images/6826673-936a42dc81db60bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 实现
在实现的时候，自己写了一个很简单的扩展myclusterlayer，代码如下：
```js
var myClusterLayer = function (options) {
    var self = this;
    var defaults = {
        map: null,
        clusterField: "",
        zooms: [4, 8, 12],
        distance: 256,
        data: [],
        style: null
    };

    //将default和options合并
    self.options = $.extend({}, defaults, options);

    self.proj = self.options.map.getView().getProjection();

    self.vectorSource = new ol.source.Vector({
        features: []
    });
    self.vectorLayer = new ol.layer.Vector({
        source: self.vectorSource,
        style:self.options.style
    });

    self.clusterData = [];

    self._clusterTest = function (data, dataCluster) {
        var _flag = false;

        var _cField = self.options.clusterField;
        if(_cField!=""){
            _flag = data[_cField] === dataCluster[_cField];
        }else{
            var _dataCoord = self._getCoordinate(data.lon, data.lat),
                _cdataCoord = self._getCoordinate(dataCluster.lon, dataCluster.lat);
            var _dataScrCoord = self.options.map.getPixelFromCoordinate(_dataCoord),
                _cdataScrCoord = self.options.map.getPixelFromCoordinate(_cdataCoord);

            var _distance = Math.sqrt(
                Math.pow((_dataScrCoord[0] - _cdataScrCoord[0]), 2) +
                Math.pow((_dataScrCoord[1] - _cdataScrCoord[1]), 2)
            );
            _flag = _distance<=self.options.distance;
        }

        var _zoom = self.options.map.getView().getZoom(),
            _maxZoom = self.options.zooms[self.options.zooms.length - 1];
        if(_zoom>_maxZoom) _flag = false;
        return _flag;
    };

    self._getCoordinate = function (lon, lat) {
        return ol.proj.transform([parseFloat(lon), parseFloat(lat)],
            "EPSG:4326",
            self.proj
        );
    };

    self._add2CluserData = function (index, data) {
        self.clusterData[index].cluster.push(data);
    };

    self._clusterCreate = function (data) {
        self.clusterData.push({
            data: data,
            cluster: []
        });
    };

    self._showCluster = function () {
        self.vectorSource.clear();
        var _features = [];
        for(var i=0,len=self.clusterData.length;i<len;i++){
            var _cdata = self.clusterData[i];
            var _coord = self._getCoordinate(_cdata.data.lon, _cdata.data.lat);
            var _feature = new ol.Feature({
                geometry: new ol.geom.Point(_coord),
                attribute: _cdata
            });
            if(_cdata.cluster.length===0) _feature.attr = _feature.data;
            _features.push(_feature);
        }
        self.vectorSource.addFeatures(_features);
    };

    self._clusterFeatures = function () {
        self.clusterData = [];
        var _viewExtent = self.options.map.getView().calculateExtent();
        var _viewGeom = new ol.geom.Polygon.fromExtent(_viewExtent);
        for(var i=0, ilen=self.options.data.length;i<ilen;i++) {
            var _data = self.options.data[i];
            var _coord = self._getCoordinate(_data.lon, _data.lat);
            if(_viewGeom.intersectsCoordinate(_coord)) {
                var _clustered = false;
                for (var j = 0, jlen = self.clusterData.length; j < jlen; j++) {
                    var _cdata = self.clusterData[j];
                    if (self._clusterTest(_data, _cdata.data)) {
                        self._add2CluserData(j, _data);
                        _clustered = true;
                        break;
                    }
                }

                if (!_clustered) {
                    self._clusterCreate(_data);
                }
            }
        }

        self.vectorSource.clear();
        self._showCluster();
    };

    self.init = function () {
        self._clusterFeatures();
        self.options.map.on("moveend", function () {
            self._clusterFeatures();
        });
    };
    self.init();

    return self.vectorLayer;
};
```
**说明**：
1、传递参数
>map: ol的map对象；
clusterField: 如果是基于属性做聚类的话可设置此参数；
zooms： 只用到了最后一个级别，当地图大于最大最后一个值的时候，全部展示；
distance：屏幕上的聚类距离；
data：聚类的数据；
style：样式（组）或者样式函数

2、核心方法
>_clusterTest：判断是否满足聚类的条件，满足则执行_add2CluserData，不满足则执行_clusterCreate；
_showCluster：展示聚类结果；

调用代码如下：
```js
                var mycluster = new myClusterLayer({
                    map: map,
                    clusterField: "",
                    zooms: [8, 11, 14],
                    distance: 100,
                    data: result,
                    style: styleFunc
                });
                map.addLayer(mycluster);
		function styleFunc(feat) {
            var attribute = feat.get("attribute");
            var count = attribute.cluster.length;
            if(count<1){
                var name = attribute.data.name;
                return new ol.style.Style({
                    image: new ol.style.Circle({
                        radius: 4,
                        fill: new ol.style.Fill({
                            color: 'red'
                        })
                    }),
                    text: new ol.style.Text({
                        text:name,
                        fill: new ol.style.Fill({
                            color: '#000000'
                        }),
                        textAlign:"left",
                        offsetX: 15,
                        textBaseline:"middle"
                    })
                })
			}else{
                var radius = Math.max(8, Math.min(count * 0.75, 20));
                return new ol.style.Style({
                    image: new ol.style.Circle({
                        radius: radius,
                        fill: new ol.style.Fill({
                            color: '#cccccc'
                        }),
                        stroke: new ol.style.Stroke({
                            color: '#ff0000',
                            width: 1.5
                        })
                    }),
                    text: new ol.style.Text({
                        text:count.toString(),
                        fill: new ol.style.Fill({
                            color: '#000000'
                        }),
                        textAlign:"center",
                        textBaseline:"middle"
                    })
                })
			}
        }
```