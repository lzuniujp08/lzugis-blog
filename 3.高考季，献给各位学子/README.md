# 概述
今天，举国瞩目的高考已经结束了，在这样的时刻“LZUGIS”携手“GIS讲堂”为大家从GIS和数据方面给大家做一个分析。

# 数据来源
数据源自**[中华人民共和国教育部](http://www.moe.gov.cn/srcsite/A03/moe_634/201706/t20170614_306900.html)**2017年06月14日生成的**全国高等学校名单**，是一个Excel的数据，数据截图如下：
![数据截图](https://upload-images.jianshu.io/upload_images/6826673-6d0e1496d2a18eb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 数据处理
拿到这样的数据肯定是没法直接用的了，为了能让数据用起来，按照如下流程做了简单的处理：
![数据处理流程](https://upload-images.jianshu.io/upload_images/6826673-a11d63b4c134094a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.根据名称添加经纬度
```java
    public String[] getLonLatByName(String name){
        String[] lonlat = new String[]{"99","99"};
        StringBuffer url = new StringBuffer();
        url.append("http://api.tianditu.com/apiserver/ajaxproxy?proxyReqUrl=")
                .append("http://map.tianditu.com/query.shtml?postStr={'keyWord':'"+name+"',")
                .append("'level':'9','mapBound':'114.6089,39.5392,118.7040,40.9562','queryType':'7','start':'0','count':'1'}&type=query");
        InputStream is = null;
        try {
            is = new URL(url.toString()).openStream();
            BufferedReader rd = new BufferedReader(new InputStreamReader(is, Charset.forName("UTF-8")));
            StringBuilder sb = new StringBuilder();
            int cp;
            while ((cp = rd.read()) != -1) {
                sb.append((char) cp);
            }
            String strJson = sb.toString().substring(19,sb.toString().length()-1);
            JSONObject json = new JSONObject(strJson);
            com.amazonaws.util.json.JSONArray arr = new com.amazonaws.util.json.JSONArray();
            if(!json.isNull("pois")){
                arr = json.getJSONArray("pois");
                JSONObject poiinfo = (JSONObject) arr.get(0);
                lonlat = poiinfo.get("lonlat").toString().split(" ");
                is.close();
            }
        }
        catch (IOException | JSONException e) {
            e.printStackTrace();
        }
        return lonlat;
    }
```
**说明**：
1、根据名称查找经纬度的过程是一个地理编码的过程，本文调用了天地图的API进行的处理；

#### 2.与行政区划做关联
跟行政区划做关联是根据经纬度给每个数据附上一个省名称的属性，这个是通过PG的空间库来实现的。
```sql
update universities set province=(select name from province where st_width(universities.geom, province.geom))
```
处理后的数据如下：
![处理后的数据](https://upload-images.jianshu.io/upload_images/6826673-dd87411eee1085cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 数据展示与分析
>将处理好的数据导出为csv文件，在GeoHey云上进行数据的展示。
#### 1. 分布散点图
![高校分布图](https://upload-images.jianshu.io/upload_images/6826673-462f251c0f142ac5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 2. 分布热力图
![高校分布热力图](https://upload-images.jianshu.io/upload_images/6826673-39b608df9577a7cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**看图说话**：
1、从分布上来看，以西安为中心，西部高校寥寥无几，除了兰州、乌鲁木齐、拉萨等省会城市，东部高校比较多也比较集中，几个比较密集的省份北京、浙江、江苏；
2、省会城市分布比较多，同时也说明了省会城市的文化中心的特点；

#### 3. 综合、本科、专科
![各省本科学校](https://upload-images.jianshu.io/upload_images/6826673-4cd7a53e7efbc3e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![各省本科统计](https://upload-images.jianshu.io/upload_images/6826673-62b9ae321b6b8925.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![各省专科学校](https://upload-images.jianshu.io/upload_images/6826673-0efdf5143e0252ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![各省专科统计](https://upload-images.jianshu.io/upload_images/6826673-44f75ceaf8d7a725.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![各省统计](https://upload-images.jianshu.io/upload_images/6826673-cfbc03dd546c705b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/6826673-515ba23edd22f526.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**说明**：
1、从本科高校分布来说，前三为江苏、北京、湖北，从专科分布来说，前三分别为江苏、广东、山东，综合来看，江苏、山东、广东为高校数量的前三；
2、西北5省+海南是垫底的，从本科高校分布来说，后三为西藏、青海、海南，从专科分布来说，后三分别为西藏、青海、海南，综合来看，西藏、青海、海南为高校数量的后三；
3、东西、南北教育资源分布的不均匀。

#### 4. 其他
![性质](https://upload-images.jianshu.io/upload_images/6826673-5c607437b0a71448.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![性质统计](https://upload-images.jianshu.io/upload_images/6826673-d91b8177f5c99612.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![类别](https://upload-images.jianshu.io/upload_images/6826673-5aac499303dd43b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![类别统计](https://upload-images.jianshu.io/upload_images/6826673-2a02e26994213424.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**说明**：
1、不难看出，在中国国立还是占了大部分的，占了73%，其余为民办和中外合作的；
2、经过处理后，参与本次统计的高校总数为2434，其中：本科院校1171所，专科院校1263所。

##### 数据下载地址：链接：https://pan.baidu.com/s/1q631PI9YJFr9UUdkZXXnTA 密码：6557
----------
**技术博客**
CSDN：http://blog.csdn.NET/gisshixisheng
**在线教程**
https://edu.csdn.net/course/detail/799
https://edu.csdn.net/course/detail/7471
**联系方式**

|类型|内容|
| :-------------: |:-------------:|
|qq|1004740957|
|公众号|lzugis15|
|e-mail|niujp08@qq.com|
|webgis群|452117357|
|Android群|337469080|
|GIS数据可视化群|458292378|

“GIS讲堂”知识星球今天开通了，在星球，我将提供一对一的问答服务，你问我答，期待与你相见。
![知识星球二维码](https://upload-images.jianshu.io/upload_images/6826673-b3d8b806aa357e49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![LZUGIS](http://upload-images.jianshu.io/upload_images/6826673-2444696a78e7b7b8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)