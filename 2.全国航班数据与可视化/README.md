# 概述
本文通过爬取全国航班的数据，并对行数据进行可视化展示与分析。

# 数据来源
数据是从哪儿也不想去的**去哪网**抓过来。为了能够获取到数据，抓取了下请求的地址，抓取的地址如下：
```
https://flight.qunar.com/touch/api/domestic/wbdflightlist?departureCity=%E5%8C%97%E4%BA%AC&arrivalCity=%E6%B7%B1%E5%9C%B3&departureDate=2018-06-12&ex_track=&__m__=09de50bb5812f2686e2db9b0f49185d0&sort=
```
返回后的数据格式如下：
![](https://upload-images.jianshu.io/upload_images/6826673-21fbbbe1e6885cda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
有了上面的URL和结果，并添加了百度地图的地理编码接口，最终得到数据如下：
![](https://upload-images.jianshu.io/upload_images/6826673-bca2655cfc3c26aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 数据处理
### 1. 数据入库
为了更好地服务于我，首先将数据做了入库（此处我用的是Postgres库），表结构如下：
```sql
CREATE TABLE domestic_flight
(
  departure_city character varying(30),
  departure_cy double precision,
  departure_cx double precision,
  landing_city character varying(30),
  landing_cy double precision,
  landing_cx double precision,
  mileage character varying(255),
  flight_schedules character varying(255),
  airlines character varying(255),
  aircraft_models character varying(255),
  departure_time character varying(255),
  landing_time character varying(255),
  departure_airport character varying(255),
  departure_y double precision,
  departure_x double precision,
  landing_airport character varying(255),
  landing_y double precision,
  landing_x double precision,
  punctuality_rate character varying(255),
  average_delayed character varying(255),
  is_mon smallint,
  is_tue smallint,
  is_wed smallint,
  is_thr smallint,
  is_fri smallint,
  is_sat smallint,
  is_sun smallint
)
```
**说明：**
1、departure为起飞，landing为落地；
2、is_sat为“sat”是否有班次，”sat”为星期几的简写；

### 2. 提取城市和机场数据
城市和机场数据是类似的，此处以机场数据为例说明。
##### 2.1 新建机场表
```sql
CREATE TABLE flight_airport
(
   id serial,
   name character varying(30),
   province character varying(30),
   airport_x numeric,
   airport_y numeric,
   CONSTRAINT pkey_air_airporty PRIMARY KEY (id)
)
```
##### 2.2 插入数据
由于机场包含起飞和落地两个类型的，所以索性将两个数据做个union。
```sql
INSERT INTO flight_airport (NAME, airport_x, airport_y)(
	SELECT DISTINCT
		departure_airport AS airport,
		departure_x AS airport_x,
		departure_y AS airport_y
	FROM
		domestic_flight
)
UNION
	(
		SELECT DISTINCT
			landing_airport AS airport,
			to_number(landing_x, '999.999999999') AS airport_x,
			to_number(landing_y, '999.999999999') AS airport_y
		FROM
			domestic_flight
	)
```
**说明：**
1、在入库的时候，没注意，将landing_x和landing_y字段类型设成了String，所以此处做了一个转换；
##### 2.3 更新省属性
省属性是通过空间表‘province’和flight_airport 表做空间关联而得的。
```sql
UPDATE flight_airport
SET province = (
	SELECT
		NAME
	FROM
		province
	WHERE
		st_within (
			st_point (
				flight_airport.airport_x,
				flight_airport.airport_y
			),
			province.geom
		)
)
```
ok， 大工告成，最后的数据如下：
![机场](https://upload-images.jianshu.io/upload_images/6826673-070984d76b0b2e0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![有机场的城市](https://upload-images.jianshu.io/upload_images/6826673-f139eefee009baa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 数据展示与分析
数据的地图展示是在**geohey云平台**上实现。
### 1.位置分布
##### 1.1 有机场的城市的位置分布
![有机场的城市](https://upload-images.jianshu.io/upload_images/6826673-026831cf697be879.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 1.2 机场的位置分布
![机场位置](https://upload-images.jianshu.io/upload_images/6826673-e4bc01852ce855f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![机场分布热力图](https://upload-images.jianshu.io/upload_images/6826673-dab41b29dfc23a3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.数据统计
##### 2.1 省机场数排名
```sql
SELECT
	province,
	COUNT (1) AS airport_num
FROM
	flight_airport
GROUP BY
	province
ORDER BY
	airport_num DESC
```
| 省名称        | 机场个数           |
| ------------- | -----:|
内蒙古    |16     |
新疆    |15     |
云南    |13     |
江苏    |11     |
四川    |11     |
山东    |11     |
甘肃    |10     |
广东    |9     |
黑龙江    |9     |
贵州    |8     |
辽宁    |8     |
浙江    |8     |
湖南    |7     |
湖北    |6     |
西藏    |6     |
福建    |6     |
广西    |6     |
安徽    |6     |
江西    |5     |
上海    |5     |
陕西    |5     |
山西    |5     |
北京    |5     |
吉林    |4     |
河北    |4     |
宁夏    |4     |
河南    |3     |
青海    |3     |
重庆    |2     |
海南    |2     |
天津    |1     |

##### 2.1 航空公司航班数排名
```sql
SELECT
	airlines as name,
	COUNT (1) AS num
FROM
	domestic_flight
GROUP BY
	airlines
ORDER BY
	num DESC
```
| 航空公司        | 航班数        |
| ------------- | -----:|
南方航空    |2552     |
东方航空    |1862     |
中国国航    |1440     |
深圳航空    |1183     |
厦门航空    |1000     |
海南航空    |982     |
山东航空    |779     |
华夏航空    |565     |
四川航空    |468     |
天津航空    |463     |
祥鹏航空    |332     |
春秋航空    |298     |
河北航空    |292     |
吉祥航空    |278     |
首都航空    |268     |
昆明航空    |239     |
成都航空    |219     |
上海航空    |214     |
西藏航空    |212     |
幸福航空    |195     |
长龙航空    |167     |
东海航空    |133     |
北部湾航空    |111     |
西部航空    |92     |
奥凯航空    |90     |
福州航空    |68     |
澳洲航空    |67     |
九元航空    |60     |
多彩航空    |52     |
新西兰航空    |52     |
大新华航空    |50     |
瑞丽航空    |46     |
扬子江航空    |42     |
青岛航空    |40     |
乌鲁木齐航空    |30     |
英国航空    |26     |
全日空航空    |18     |
重庆航空    |17     |
香港航空    |17     |
红土航空    |13     |
日本航空    |13     |
联合航空    |8     |
桂林航空    |4     |
江西航空    |4     |
夏威夷航空    |3     |
酷航    |3     |
长安航空    |2     |
北欧航空    |2     |
##### 2.3 飞机机型排名
```sql
SELECT
	aircraft_models as name,
	COUNT (1) AS num
FROM
	domestic_flight
GROUP BY
	aircraft_models
ORDER BY
	num DESC
```
| 机型名称       | 个数        |
| ------------- | -----:|
波音737(中)    |6243     |
空客320(中)    |3697     |
空客319(中)    |1278     |
JET    |987     |
空客321(中)    |787     |
ERJ-190(中)    |678     |
庞巴迪CRJ900    |547     |
其他机型    |457     |
新舟60(小)    |145     |
空客330(宽体机)    |108     |
空客321(窄体机)    |37     |
波音787(大)    |30     |
CRJ(小)    |24     |
ERJ(小)    |20     |
波音777(大)    |15     |
波音757(中)    |10     |
空客380(大)    |5     |
波音747(大)    |2     |
波音767(大)    |1     |
##### 数据下载地址：链接：https://pan.baidu.com/s/1FQW1OkNtALIvfoxFbcn4vw 密码：m1k2