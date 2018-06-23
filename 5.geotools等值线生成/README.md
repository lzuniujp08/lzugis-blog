# 概述
前文中，提到了[等值面](https://www.jianshu.com/p/78040229d47a)的生成,后面有人经常会问等值线的生成，本文在前文的基础上做了一点修改，完成了等值线的geotools生成。

# 效果
![未裁剪](https://upload-images.jianshu.io/upload_images/6826673-fe9b9672af1adf78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![裁剪后](https://upload-images.jianshu.io/upload_images/6826673-51e9f9fab0c59a0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 实现代码
```java
package com.lzugis.geotools;

import com.amazonaws.util.json.JSONObject;
import com.lzugis.CommonMethod;
import com.lzugis.geotools.utils.FeaureUtil;
import com.lzugis.geotools.utils.GeoJSONUtil;
import com.vividsolutions.jts.geom.Geometry;
import org.geotools.data.FeatureSource;
import org.geotools.data.shapefile.ShapefileDataStore;
import org.geotools.data.simple.SimpleFeatureCollection;
import org.geotools.data.simple.SimpleFeatureSource;
import org.geotools.feature.FeatureCollection;
import org.geotools.feature.FeatureIterator;
import org.geotools.geojson.feature.FeatureJSON;
import org.opengis.feature.Feature;
import org.opengis.feature.simple.SimpleFeature;
import wContour.Contour;
import wContour.Global.Border;
import wContour.Global.PointD;
import wContour.Global.PolyLine;
import wContour.Global.Polygon;
import wContour.Interpolate;

import java.io.File;
import java.io.StringWriter;
import java.nio.charset.Charset;
import java.util.*;

/**
 * Created by admin on 2017/8/29.
 */
public class EquiSurfaceLine {
    private static String rootPath = System.getProperty("user.dir");

    /**
     * 生成等值面
     *
     * @param trainData    训练数据
     * @param dataInterval 数据间隔
     * @param size         大小，宽，高
     * @param boundryFile  四至
     * @param isclip       是否裁剪
     * @return
     */
    public String calEquiSurface(double[][] trainData,
                                 double[] dataInterval,
                                 int[] size,
                                 String boundryFile,
                                 boolean isclip) {
        String geojsonline = "";
        try {
            double _undefData = -9999.0;
            SimpleFeatureCollection polylineCollection = null;
            List<PolyLine> cPolylineList = new ArrayList<PolyLine>();
            List<Polygon> cPolygonList = new ArrayList<Polygon>();

            int width = size[0],
                    height = size[1];
            double[] _X = new double[width];
            double[] _Y = new double[height];

            File file = new File(boundryFile);
            ShapefileDataStore shpDataStore = null;

            shpDataStore = new ShapefileDataStore(file.toURL());
            //设置编码
            Charset charset = Charset.forName("GBK");
            shpDataStore.setCharset(charset);
            String typeName = shpDataStore.getTypeNames()[0];
            SimpleFeatureSource featureSource = null;
            featureSource = shpDataStore.getFeatureSource(typeName);
            SimpleFeatureCollection fc = featureSource.getFeatures();

            double minX = fc.getBounds().getMinX();
            double minY = fc.getBounds().getMinY();
            double maxX = fc.getBounds().getMaxX();
            double maxY = fc.getBounds().getMaxY();

            Interpolate.CreateGridXY_Num(minX, minY, maxX, maxY, _X, _Y);
            double[][] _gridData = new double[width][height];

            int nc = dataInterval.length;

            _gridData = Interpolate.Interpolation_IDW_Neighbor(trainData,
                    _X, _Y, 12, _undefData);// IDW插值

            int[][] S1 = new int[_gridData.length][_gridData[0].length];
            /**
             * double[][] S0,
             * double[] X,
             * double[] Y,
             * int[][] S1,
             * double undefData
             */
            List<Border> _borders = Contour.tracingBorders(_gridData, _X, _Y,
                    S1, _undefData);

            /**
             * double[][] S0,
             * double[] X,
             * double[] Y,
             * int nc,
             * double[] contour,
             * double undefData,
             * List<Border> borders,
             * int[][] S1
             */
            cPolylineList = Contour.tracingContourLines(_gridData, _X, _Y, nc,
                    dataInterval, _undefData, _borders, S1);// 生成等值线

            cPolylineList = Contour.smoothLines(cPolylineList);// 平滑

            geojsonline = getPolylineGeoJson(cPolylineList);

            if (isclip) {
                polylineCollection = GeoJSONUtil.readGeoJsonByString(geojsonline);
                FeatureSource dc = clipFeatureCollection(fc, polylineCollection);
                geojsonline = getPolylineGeoJson(dc.getFeatures());
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
        return geojsonline;
    }


    public String getPolylineGeoJson(FeatureCollection fc) {
        FeatureJSON fjson = new FeatureJSON();
        StringBuffer sb = new StringBuffer();
        try {
            sb.append("{\"type\": \"FeatureCollection\",\"features\": ");
            FeatureIterator itertor = fc.features();
            List<String> list = new ArrayList<String>();
            while (itertor.hasNext()) {
                SimpleFeature feature = (SimpleFeature) itertor.next();
                StringWriter writer = new StringWriter();
                fjson.writeFeature(feature, writer);
                list.add(writer.toString());
            }
            itertor.close();
            sb.append(list.toString());
            sb.append("}");
        } catch (Exception e) {
            e.printStackTrace();
        }
        return sb.toString();
    }

    public FeatureSource clipFeatureCollection(FeatureCollection fc,
                                               SimpleFeatureCollection gs) {
        FeatureSource cs = null;
        try {
            List<Map<String, Object>> values = new ArrayList<Map<String, Object>>();
            FeatureIterator contourFeatureIterator = gs.features();
            FeatureIterator dataFeatureIterator = fc.features();
            while (dataFeatureIterator.hasNext()) {
                Feature dataFeature = dataFeatureIterator.next();
                Geometry dataGeometry = (Geometry) dataFeature.getProperty(
                        "the_geom").getValue();
                while (contourFeatureIterator.hasNext()) {
                    Feature contourFeature = contourFeatureIterator.next();
                    Geometry contourGeometry = (Geometry) contourFeature
                            .getProperty("geometry").getValue();
                    double v = (Double) contourFeature.getProperty("value")
                            .getValue();
                    if (dataGeometry.intersects(contourGeometry)) {
                        Geometry geo = dataGeometry
                                .intersection(contourGeometry);
                        Map<String, Object> map = new HashMap<String, Object>();
                        map.put("the_geom", geo);
                        map.put("value", v);
                        values.add(map);
                    }

                }

            }

            contourFeatureIterator.close();
            dataFeatureIterator.close();

            SimpleFeatureCollection sc = FeaureUtil
                    .creatSimpleFeatureByFeilds(
                            "polygons",
                            "crs:4326,the_geom:LineString,value:double",
                            values);
            cs = FeaureUtil.creatFeatureSourceByCollection(sc);

        } catch (Exception e) {
            e.printStackTrace();
            return cs;
        }

        return cs;
    }


    public String getPolylineGeoJson(List<PolyLine> cPolylineList) {
        String geo = null;
        String geometry = " { \"type\":\"Feature\",\"geometry\":";
        String properties = ",\"properties\":{ \"value\":";

        String head = "{\"type\": \"FeatureCollection\"," + "\"features\": [";
        String end = "  ] }";
        if (cPolylineList == null || cPolylineList.size() == 0) {
            return null;
        }
        try {
            for (PolyLine pPolyline : cPolylineList) {
                List<Object> ptsTotal = new ArrayList<Object>();

                for (PointD ptD : pPolyline.PointList) {
                    List<Double> pt = new ArrayList<Double>();
                    pt.add(ptD.X);
                    pt.add(ptD.Y);
                    ptsTotal.add(pt);
                }

                JSONObject js = new JSONObject();
                js.put("type", "LineString");
                js.put("coordinates", ptsTotal);

                geo = geometry + js.toString() + properties + pPolyline.Value + "} }" + "," + geo;
            }
            if (geo.contains(",")) {
                geo = geo.substring(0, geo.lastIndexOf(","));
            }

            geo = head + geo + end;
        } catch (Exception e) {
            e.printStackTrace();
            return geo;
        }
        return geo;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();

        EquiSurfaceLine equiSurface = new EquiSurfaceLine();
        CommonMethod cm = new CommonMethod();

        double[] bounds = {110.759, 31.23, 113.112, 32.6299};

        double[][] trainData = new double[3][100];

        for (int i = 0; i < 100; i++) {
            double x = bounds[0] + new Random().nextDouble() * (bounds[2] - bounds[0]),
                    y = bounds[1] + new Random().nextDouble() * (bounds[3] - bounds[1]),
                    v = 0 + new Random().nextDouble() * (45 - 0);
            trainData[0][i] = x;
            trainData[1][i] = y;
            trainData[2][i] = v;
        }

        double[] dataInterval = new double[]{20, 25, 30, 35, 40, 45};

        String boundryFile = rootPath + "/data/shp/XY_01_XZQH_PY.shp";

        int[] size = new int[]{100, 100};

        boolean isclip = true;

        try {
            String strJson = equiSurface.calEquiSurface(trainData, dataInterval, size, boundryFile, isclip);
            String strFile = rootPath + "/out/XY_01_XZQH_PY_line1.geojson";
            cm.append2File(strFile, strJson);
            System.out.println(strFile + "差值成功, 共耗时" + (System.currentTimeMillis() - start) + "ms");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```