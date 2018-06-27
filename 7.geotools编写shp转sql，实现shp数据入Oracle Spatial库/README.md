# 概述
用到Oracle Spatial就很难避免shp文件的入库问题，虽然有shp2sdo工具，但是用起来不是很习惯，所以，本文讲述如何结合geotools实现shp2sql的转换。

# 效果
![效果](https://upload-images.jianshu.io/upload_images/6826673-b6e12576b6d8935d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 实现代码
```java
package com.lzugis.geotools;

import com.lzugis.CommonMethod;
import com.vividsolutions.jts.geom.Coordinate;
import com.vividsolutions.jts.geom.Geometry;
import com.vividsolutions.jts.geom.GeometryFactory;
import com.vividsolutions.jts.simplify.TopologyPreservingSimplifier;
import org.geotools.data.shapefile.ShapefileDataStore;
import org.geotools.data.simple.SimpleFeatureCollection;
import org.geotools.data.simple.SimpleFeatureIterator;
import org.geotools.data.simple.SimpleFeatureSource;
import org.geotools.factory.CommonFactoryFinder;
import org.geotools.filter.text.cql2.CQL;
import org.opengis.feature.simple.SimpleFeature;
import org.opengis.feature.simple.SimpleFeatureType;
import org.opengis.feature.type.AttributeDescriptor;
import org.opengis.feature.type.AttributeType;
import org.opengis.filter.Filter;
import org.opengis.filter.FilterFactory2;
import scala.util.parsing.combinator.testing.Str;

import java.io.File;
import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import com.lzugis.geotools.model.Fields;

public class Shp2Orcale {
	private final Map<String, String> fMap = new HashMap<>();

	private SimpleFeatureSource featureSource = null;
	private String tableName = "";
	private List<Fields> fields = new ArrayList<>();
	private CommonMethod cm = new CommonMethod();

	public Shp2Orcale(){
		fMap.put("string", "varchar2(64)");
		fMap.put("long", "long");
		fMap.put("double", "number");
		fMap.put("shape", "\"MDSYS\".\"SDO_GEOMETRY\"");
	}

	public void readShape(String shpfile){
		try {
			File file = new File(shpfile);
			ShapefileDataStore shpDataStore = null;

			shpDataStore = new ShapefileDataStore(file.toURL());
			//设置编码
			Charset charset = Charset.forName("GBK");
			shpDataStore.setCharset(charset);
			tableName = shpDataStore.getTypeNames()[0];
			featureSource =  shpDataStore.getFeatureSource (tableName);
		}catch (Exception e){
			e.printStackTrace();
		}
	}

	public void getShpFields(){
		SimpleFeatureType schema = featureSource.getSchema();
		List<AttributeDescriptor> attrs= schema.getAttributeDescriptors();
		for(int i=0;i<attrs.size();i++){
			AttributeDescriptor attr = attrs.get(i);
			Class<?> cls = attr.getType().getBinding();
			String clsName = cls.getName();
			clsName = clsName.substring(clsName.lastIndexOf(".")+1).toLowerCase();
			Fields field = new Fields(attr.getLocalName(), clsName);
			fields.add(field);
		}
	}

	public void createTableSql(String sqlpath){
		cm.append2File(sqlpath, "--1.drop table\r\nDROP TABLE "+tableName+";");
		cm.append2File(sqlpath, "\r\n--2.create table\r\n");
		StringBuffer sql = new StringBuffer();
		sql.append("CREATE TABLE "+tableName+" (");
		for(int i=0, size = fields.size();i<size;i++){
			Fields field = fields.get(i);
			String filedname = field.getFieldmame().toLowerCase();
			filedname = filedname.equals("the_geom")?"shape":filedname;
			String fieldtype = field.getFieldtype();
			fieldtype = filedname.equals("shape")?"shape":fieldtype;
			sql.append(filedname+" "+fMap.get(fieldtype));
			if(i!=size-1) sql.append(", ");
		}
		sql.append(");");
		cm.append2File(sqlpath, sql.toString());
	}

	public void insertValueSql(String sqlpath) {
		try {
			cm.append2File(sqlpath, "\r\n--3.insert data");

			SimpleFeatureCollection result = featureSource.getFeatures();
			SimpleFeatureIterator itertor = result.features();

			StringBuffer sql = new StringBuffer();
			while (itertor.hasNext()){
				SimpleFeature feature = itertor.next();
				StringBuffer _sql = new StringBuffer();
				_sql.append("insert into "+tableName+" values(");
				for(int i=0, size = fields.size();i<size;i++){
					Fields field = fields.get(i);
					String filedname = field.getFieldmame();
					filedname = filedname=="the_geom"?"shape":filedname;
					String fieldtype = field.getFieldtype();

					if(filedname!="shape"){
						if(fieldtype.equals("string")){
							_sql.append("'"+feature.getAttribute(filedname)+"'");
						}else {
							_sql.append(feature.getAttribute(filedname));
						}
					}else{
						Geometry geom = (Geometry)feature.getAttribute("the_geom");
						_sql.append("sdo_geometry('"+geom.toString()+"', 4326)");
					}
					if(i!=size-1) _sql.append(", ");
				}
				_sql.append(");\r\n");
				sql.append(_sql);
			}
			cm.append2File(sqlpath, sql.toString());
		}catch (Exception e){
			e.printStackTrace();
		}
	}

	/**
	 * 工具类测试方法
	 * @param args
	 */
	public static void main(String[] args) {
		Shp2Orcale shp2orcl =new Shp2Orcale();
		String shppath = "D:\\data\\wgs84\\capital.shp";
		String sqlpath = "D:\\data\\wgs84\\sql\\capital.sql";
        try{
    		long start = System.currentTimeMillis();
    		//读取shp文件
			shp2orcl.readShape(shppath);
			shp2orcl.getShpFields();
			shp2orcl.createTableSql(sqlpath);
			shp2orcl.insertValueSql(sqlpath);
    		System.out.println("共耗时"+(System.currentTimeMillis() - start)+"ms");
        }
        catch(Exception e){
        	e.printStackTrace();
        }
	}
}
```
**说明：**
1、对于比较复杂的线或面会出现sql太长的问题，文章里面加了一行简化的代码；