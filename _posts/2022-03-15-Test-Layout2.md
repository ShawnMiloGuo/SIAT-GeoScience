---
layout: article
title: Test layout 2
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/image-20220304100432-74yo59i.png
---

寻找实验合适的MODIS和Landsat数据集

# 地表温度数据

## Landsat 8

landsat 有Collection 1 和Collection 2 两个数据集， Collection 2 比较处理算法比较新一些，但数量可能少一些。

每个Collection中数据被分为三个级别，RealTime，Tier1 和Tier 2

[Page layout](https://tianqi.name/jekyll-TeXt-theme/samples.html#page-layout)

[Landsat Collections — What are Tiers? , U.S. Geological Survey (usgs.gov)](https://www.usgs.gov/media/videos/landsat-collections-what-are-tiers)

* Real Time： 数据在卫星上获取后，进行快速的处理，用于对突发事件的实时监测
* Tier 1 数据是RMSE小于12 meters 的数据，为了构建准确的时间序列数据
* ![image.png](/assets/image-20220303175951-jezavw8.png){:.rounded}
  Tier 2 数据是RMSE 大于12 meters的数据，可以说是有问题的数据，这些数据定位差主要是因为云的问题
  ![image.png](/assets/image-20220303175914-ev5h3qo.png){:.rounded}

### USGS Landsat 8 Level 2, Collection 2, Tier 2

[USGS Landsat 8 Level 2, Collection 2, Tier 2  ,  Earth Engine Data Catalog  ,  Google Developers](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T2_L2#bands)

```javascript
var geometry = 
    /* color: #98ff00 */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[113.83369619704567, 22.638199388931643],
          [113.83369619704567, 22.540251386157504],
          [114.01771719313942, 22.540251386157504],
          [114.01771719313942, 22.638199388931643]]], null, false);

var dataset = ee.ImageCollection('LANDSAT/LC08/C02/T2_L2')
    .filterDate('2014-01-01', '2022-12-31').filterBounds(geometry).sort('DATE_ACQUIRED');

print(dataset)
// Applies scaling factors.
function applyScaleFactors(image) {
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBands = image.select('ST_B.*').multiply(0.00341802).add(149.0);
  return image.addBands(opticalBands, null, true)
              .addBands(thermalBands, null, true);
}

dataset = dataset.map(applyScaleFactors).select('ST_B10');

var visualization = {
  bands: ['ST_B10'],
  min: 200,
  max: 323,
};

Map.addLayer(dataset.max(), visualization, 'True Color (432)');
```

发现该在深圳区域，同一path row 下数据时间序列不完整。从2014年到现在的数据列表如下：

![image.png](/assets/image-20220303154333-jrmfxs3.png#pic_center){:.rounded}

通过筛选Cloud Cover小于 50% 的只有两景

```javascript
var dataset = ee.ImageCollection('LANDSAT/LC08/C02/T2_L2')
    .filterDate('2014-01-05', '2022-05-09').filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lte('CLOUD_COVER_LAND',50))

```

![image.png](/assets/image-20220303162321-5uxiehe.png#pic_center)

`LANDSAT/LC08/C02/T2_L2/LC08_122044_20160310`

`LANDSAT/LC08/C02/T2_L2/LC08_122044_20170617`

实际上全是云

![image.png](/assets/image-20220303165544-r47xwna.png){:.rounded}

![image.png](/assets/image-20220303165827-40zov6m.png){:.rounded}

小结：LANDSAT/LC08/C02/T2_L2这个数据集不能用


### USGS Landsat 8 Surface Reflectance Tier 1

```javascript
var geometry = 
    /* color: #0b4a8b */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[113.94192856337884, 22.639014162386058],
          [113.94192856337884, 22.58830574704232],
          [114.04286545302728, 22.58830574704232],
          [114.04286545302728, 22.639014162386058]]], null, false);
/**
 * Function to mask clouds based on the pixel_qa band of Landsat 8 SR data.
 * @param {ee.Image} image Input Landsat 8 SR image
 * @return {ee.Image} Cloudmasked Landsat 8 image
 */
function maskL8sr(image) {
  // Bits 3 and 5 are cloud shadow and cloud, respectively.
  var cloudShadowBitMask = (1 << 3);
  var cloudsBitMask = (1 << 5);
  // Get the pixel QA band.
  var qa = image.select('pixel_qa');
  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
                 .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask);
}

var dataset = ee.ImageCollection('LANDSAT/LC08/C01/T1_SR')
                  .filterDate('2014-01-01', '2022-12-31').filterBounds(geometry)
                  //.map(maskL8sr);
        
print(dataset)

var visParams = {
  bands: ['B4', 'B1', 'B2'],
  min: 0,
  max: 3000,
  gamma: 1.4,
};
Map.addLayer(dataset.median(), visParams);

```

该数据集中云量小于1的有13景

![image.png](/assets/image-20220304084539-t4vx9ln.png)

### USGS Landsat 8 Collection 1 Tier 1 and Real-Time data TOA Reflectance

[USGS Landsat 8 Collection 1 Tier 1 and Real-Time data TOA Reflectance  ,  Earth Engine Data Catalog  ,  Google Developers](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C01_T1_RT_TOA)

```javascript
var dataset = ee.ImageCollection('LANDSAT/LC08/C01/T1_RT_TOA')
                  .filterDate('2014-01-01', '2022-12-31').filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lt('CLOUD_COVER_LAND',1))
print(dataset)
var trueColor432 = dataset.select(['B4', 'B3', 'B2']);
var trueColor432Vis = {
  min: 0.0,
  max: 0.4,
};


Map.addLayer(trueColor432.mean(), trueColor432Vis, 'True Color (432)');
```

该数据集中云量小于1的有13景

![image.png](/assets/image-20220303172928-tycx7jn.png)

### USGS Landsat 8 Collection 1 Tier 1 TOA Reflectance

[USGS Landsat 8 Collection 1 Tier 1 TOA Reflectance  ,  Earth Engine Data Catalog  ,  Google Developers](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C01_T1_TOA)

该数据集和

```javascript
var dataset = ee.ImageCollection('LANDSAT/LC08/C01/T1_TOA')
                  .filterDate('2014-01-01', '2022-12-31').filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lt('CLOUD_COVER_LAND',1))
print(dataset)

var trueColor432 = dataset.select(['B4', 'B3', 'B2']);
var trueColor432Vis = {
  min: 0.0,
  max: 0.4,
};

Map.addLayer(trueColor432, trueColor432Vis, 'True Color (432)');

```

![image.png](/assets/image-20220303173848-tg8lc59.png)

### USGS Landsat 8 Collection 2 Tier 1 TOA Reflectance

[USGS Landsat 8 Collection 2 Tier 1 TOA Reflectance  ,  Earth Engine Data Catalog  ,  Google Developers](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T1_TOA)

`ee.ImageCollection("LANDSAT/LC08/C02/T1_TOA")`

```javascript
var dataset = ee.ImageCollection("LANDSAT/LC08/C02/T1_TOA")
  .filterDate('2014-01-01', '2022-06-18').filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lte('CLOUD_COVER_LAND',1))
  print(dataset)
var trueColor432 = dataset.select(['B4', 'B3', 'B2']);
var trueColor432Vis = {
  min: 0.0,
  max: 0.4,
};

Map.addLayer(trueColor432.mean(), trueColor432Vis, 'True Color (432)');
```

![image.png](/assets/image-20220304083845-bv379lf.png)

### USGS Landsat 8 Level 2, Collection 2, Tier 1

[USGS Landsat 8 Level 2, Collection 2, Tier 1  ,  Earth Engine Data Catalog  ,  Google Developers](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T1_L2#description)

`ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")`

![image.png](/assets/image-20220304085912-dwvlfhm.png)

```javascript


var dataset = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
    .filterDate('2000-01-16', '2022-06-18').filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lte('CLOUD_COVER_LAND',1))

// A function that scales and masks Landsat 8 (C2) surface reflectance images.
function prepSrL8(image) {
  // Develop masks for unwanted pixels (fill, cloud, cloud shadow).
  var qaMask = image.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
  var saturationMask = image.select('QA_RADSAT').eq(0);

  // Apply the scaling factors to the appropriate bands.
  var getFactorImg = function(factorNames) {
    var factorList = image.toDictionary().select(factorNames).values();
    return ee.Image.constant(factorList);
  };
  var scaleImg = getFactorImg([
    'REFLECTANCE_MULT_BAND_.,TEMPERATURE_MULT_BAND_ST_B10']);
  var offsetImg = getFactorImg([
    'REFLECTANCE_ADD_BAND_.,TEMPERATURE_ADD_BAND_ST_B10']);
  var scaled = image.select('SR_B.,ST_B10').multiply(scaleImg).add(offsetImg);

  // Replace original bands with scaled bands and apply masks.
  return image.addBands(scaled, null, true)
    .updateMask(qaMask).updateMask(saturationMask);
}
// Applies scaling factors.
function applyScaleFactors(image) {
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBands = image.select('ST_B.*').multiply(0.00341802).add(149.0);
  return image.addBands(opticalBands, null, true)
              .addBands(thermalBands, null, true);
}

dataset = dataset.map(prepSrL8);
print(dataset)
var visualization = {
  bands: ['ST_B10'],
  palette: [
    '040274', '040281', '0502a3', '0502b8', '0502ce', '0502e6',
    '0602ff', '235cb1', '307ef3', '269db1', '30c8e2', '32d3ef',
    '3be285', '3ff38f', '86e26f', '3ae237', 'b5e22e', 'd6e21f',
    'fff705', 'ffd611', 'ffb613', 'ff8b13', 'ff6e08', 'ff500d',
    'ff0000', 'de0101', 'c21301', 'a71001', '911003'
  ],
  min: 270,
  max: 320,
};
var visParams = {
  bands: ['SR_B4', 'SR_B3', 'SR_B2'],
  min: 0.0,
  max:0.4,
  gamma: 1.4,
};


var Layer = ee.Image(dataset.toList(20).get(0));
var Layer2013 = ee.Image(dataset.toList(20).get(0));
var Layer2021 = ee.Image(dataset.toList(20).get(14));
print(Layer)
Map.addLayer(Layer2013, visualization, 'LST-2013');
Map.addLayer(Layer2013, visParams, 'True Color (432)-2013');
Map.addLayer(Layer2021, visualization, 'LST-2021');
Map.addLayer(Layer2021, visParams, 'True Color (432)-2021');
```

由于LST 数据分辨率较低，因此需要区域更大一些

![image.png](//assets/image-20220304100400-fp4uuno.png)

![image.png](/assets/image-20220304100432-74yo59i.png)

![image.png](/assets/image-20220304100820-1ysuiwf.png)

最终选取USGS Landsat 8 Level 2, Collection 2, Tier 1


### USGS Landsat 7 Level 2, Collection 2, Tier 1

[USGS Landsat 7 Level 2, Collection 2, Tier 1  ,  Earth Engine Data Catalog  ,  Google Developers](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LE07_C02_T1_L2#bands)

`ee.ImageCollection("LANDSAT/LE07/C02/T1_L2")`

```javascript
var dataset = ee.ImageCollection('LANDSAT/LE07/C02/T1_L2')
    .filterDate('2013-12-07', '2013-12-08').filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lte('CLOUD_COVER_LAND',1))

// Applies scaling factors.
function applyScaleFactors(image) {
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBand = image.select('ST_B6').multiply(0.00341802).add(149.0);
  return image.addBands(opticalBands, null, true)
              .addBands(thermalBand, null, true);
}

dataset = dataset.map(applyScaleFactors);
print(dataset);
var visualization = {
  bands: ['SR_B3', 'SR_B2', 'SR_B1'],
  min: 0.0,
  max: 0.3,
};



Map.addLayer(dataset, visualization, 'True Color (321)');

```

![image.png](/assets/image-20220304104328-658aelh.png)

On May 31st, 2003. SLC Fail, 所以2003之前的数据可以用


## MODIS-Terra

### MOD11A1.006 Terra Land Surface Temperature and Emissivity Daily Global 1km

[MOD11A1.006 Terra Land Surface Temperature and Emissivity Daily Global 1km  ,  Earth Engine Data Catalog  ,  Google Developers](https://developers.google.com/earth-engine/datasets/catalog/MODIS_006_MOD11A1)

`ee.ImageCollection("MODIS/006/MOD11A1")`

```javascript
var geometry = 
    /* color: #d63000 */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[113.92280112535597, 22.56171618547491],
          [113.92280112535597, 22.53127630039697],
          [113.96502982408644, 22.53127630039697],
          [113.96502982408644, 22.56171618547491]]], null, false),
    landSurfaceTemperatureVis = {"palette":["040274","040281","0502a3","0502b8","0502ce","0502e6","0602ff","235cb1","307ef3","269db1","30c8e2","32d3ef","3be285","3ff38f","86e26f","3ae237","b5e22e","d6e21f","fff705","ffd611","ffb613","ff8b13","ff6e08","ff500d","ff0000","de0101","c21301","a71001","911003"]};
var dataset = ee.ImageCollection('MODIS/006/MOD11A1')
                  .filter(ee.Filter.date('2013-12-31', '2014-01-01')).filterBounds(geometry);
print(dataset);
var landSurfaceTemperature = dataset.select('LST_Day_1km');
var landSurfaceTemperatureDayviewTime = dataset.select('Day_view_time')


Map.addLayer(
    landSurfaceTemperatureDayviewTime, landSurfaceTemperatureVis,
    'Land Surface Temperature');
```

## 面临的问题

### 问题描述：

发现凡是TM8过境的那一天，深圳都会刚好在MODIS的条带中，无数据

![image.png](/assets/image-20220304102957-yogc9ga.png)

如何去解决这个问题：

1. 换试验区，将试验区移动到深圳西部？
2. 换LC8 数据为Landsat7 数据
3. 换MODIS数据？还没找到替代品
4. 用MODIS前一天或后一天的数据代替？

### 暂定的解决思路

1. 在2000-2003年的数据中选取 Landsat 7的无条带数据作为数据源
2. 在2003年以后的数据，选取Landsat 8, 但MODIS选取之前一天或之后一天的数据，这样的假设是，前一天和后一天的气温没有发生突变。
3. 换做北京区域

## 数据准备的代码：

"数据准备的代码"[^1]


[^1]: # 数据准备的代码

    ### MODIS 和Landsat 7 合成

    ```javascript
    ////////////////////////////////////////////////////////////////////////
    /*
    Copyright (c) 2022, Shanxin Guo
    All rights reserved.  
    Unauthorized reproduction prohibited.  
    This software may be used, copied, or redistributed as long as it is not  
    sold and this copyright notice is reproduced on each copy made.  This  
    routine is provided as is without any express or implied warranties  
    whatsoever. 

    PURPOSE:  
      Combine MOD11A1  (1.2km spatial resolution) with TM 7 LST Collection 2
      Cloud is removed from both TM and MODIS
      
      the output image have two bands, the first one is TM LST , the second one is MODIS LST

    Referance: 
      1.example in GEE

    MODIFICATION HISTORY:  
     Written by: Shanxin Guo , 2022-03-04, sx.guo@siat.ac.cn
    */
    //////////////////////////////////////////////////////////////////////////////////

    // Applies scaling factors.
    function applyScaleFactors_MODISLST(image) {
      var thermalBand = image.select('LST_Day_1km').multiply(0.02);
      return image.addBands(thermalBand, null, true);
    }
    // Applies scaling factors.
    function applyScaleFactors_TM7LST(image) {
      var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
      var thermalBand = image.select('ST_B6').multiply(0.00341802).add(149.0);
      return image.addBands(opticalBands, null, true)
                  .addBands(thermalBand, null, true);
    }

    var MovingWindowJoint =function(primary,secondary)
    {

      var windowSize= 0.5 * 24 * 60 * 60 * 1000;
      //Define different timeFilters according modes
      var timeFilter = ee.Filter.maxDifference({
             difference: windowSize,
             leftField: 'system:time_end',
             rightField: 'system:time_start'
            });

      //Define the join
      var saveAllJoin = ee.Join.saveAll({
         matchesKey: 'matchedID',
         ordering: 'system:time_start',
         ascending: true,
         measureKey: 'timeDiff'
      });
      
      // Apply the join.
      var joinedCollection = saveAllJoin.apply(primary, secondary, timeFilter);
      return joinedCollection;
    };

    var combineGAGQ = function(image)
    {
      var TempGQ=ee.List(image.get('matchedID'));
      var imageCollection=ee.ImageCollection(TempGQ);
      image=ee.Image(image).addBands(imageCollection.first());
      return image;
    };

    ////Clip MODIS band by TM Band
    var ClipMODISByTMBoundary =  function(image)
    {
      var TM= image.select(0);
      var MODIS=image.select(1);
      var footprint=TM.get('system:footprint');
      var coordinates =ee.Geometry(footprint).coordinates() ; 
      var boundary=ee.Geometry.Polygon(coordinates);
      MODIS=MODIS.clip(boundary);
      
      //Mask MODIS coloud with TM Mask
      MODIS=MODIS.updateMask(TM.gt(0));
      var FinalImage=ee.Image(TM).addBands(MODIS);
      return FinalImage;
    };
    //////////////////////////////////////////////////////////////////////////////////
    //Main function

    //data preparing
    var  date1 = '2000-03-01';
    var  date2 = '2003-05-30';
    var collectionGQ = ee.ImageCollection('MODIS/006/MOD11A1').filterDate(date1, date2).filterBounds(geometry);
    var GQ = collectionGQ.map(applyScaleFactors_MODISLST).select('LST_Day_1km');
    //print(GQ)

    var collectionTM7 = ee.ImageCollection('LANDSAT/LE07/C02/T1_L2').filterDate(date1, date2).filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lte('CLOUD_COVER_LAND',1));
    var TM = collectionTM7.map(applyScaleFactors_TM7LST).select('ST_B6');
    //print(TM)

    //joint TM with GQ
    var CombinationGAGQ=MovingWindowJoint(TM,GQ);
    //print(CombinationGAGQ);
    //combine to one imagecollection
    var TMGQ=CombinationGAGQ.map(combineGAGQ);
    //print(TMGQ);
    //re-arrange the bands with new name
    TMGQ=ee.ImageCollection(TMGQ).select([0,1],['TMLST','MODISLST']);
    //Clip MODIS band by TM Band
    var TMMODIS=TMGQ.map(ClipMODISByTMBoundary);
    //re-arrange the bands with new name
    TMMODIS=ee.ImageCollection(TMMODIS).select([0,1],['TMLST','MODISLST']);
    print(TMMODIS);

    var LST_PALETTE = {
      palette: [
        '040274', '040281', '0502a3', '0502b8', '0502ce', '0502e6',
        '0602ff', '235cb1', '307ef3', '269db1', '30c8e2', '32d3ef',
        '3be285', '3ff38f', '86e26f', '3ae237', 'b5e22e', 'd6e21f',
        'fff705', 'ffd611', 'ffb613', 'ff8b13', 'ff6e08', 'ff500d',
        'ff0000', 'de0101', 'c21301', 'a71001', '911003'
      ],
      min: 270,
      max: 320,
    };
    //Image index you want to display and output
    var index=2;

    var TM2 = ee.Image(TMMODIS.toList(10).get(index));
    Map.addLayer(TM2.select(0),LST_PALETTE,"TM LST");
    Map.addLayer(TM2.select(1),LST_PALETTE,"MODIS LST");

    //output to google driver
    var TMfilename=TM2.get('system:index');
    print(TMfilename);

    var footprint=TM2.get('system:footprint');
    var coordinates =ee.Geometry(footprint).coordinates() ; 
    var boundary=ee.Geometry.Polygon(coordinates);
    Export.image(TM2, TMfilename.getInfo(), {
      scale: 30, // here MODIS data will be resampled to 30m
      region:boundary
    });
    ```

    ### MODIS 和 Landsat 8 合成

    ```javascript
    ////////////////////////////////////////////////////////////////////////
    /*
    Copyright (c) 2022, Shanxin Guo
    All rights reserved.  
    Unauthorized reproduction prohibited.  
    This software may be used, copied, or redistributed as long as it is not  
    sold and this copyright notice is reproduced on each copy made.  This  
    routine is provided as is without any express or implied warranties  
    whatsoever. 

    PURPOSE:  
      Combine MOD11A1  (1.2km spatial resolution) with TM 7 LST Collection 2
      Cloud is removed from both TM and MODIS
      
      the output image have two bands, the first one is TM LST , the second one is MODIS LST

    Referance: 
      1.example in GEE

    MODIFICATION HISTORY:  
     Written by: Shanxin Guo , 2022-03-04, sx.guo@siat.ac.cn
    */
    //////////////////////////////////////////////////////////////////////////////////

    // Applies scaling factors.
    function applyScaleFactors_MODISLST(image) {
      var thermalBand = image.select('LST_Day_1km').multiply(0.02);
      return image.addBands(thermalBand, null, true);
    }
    function applyScaleFactors_TM8LST(image) {
      var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
      var thermalBand = image.select('ST_B10').multiply(0.00341802).add(149.0);
      return image.addBands(opticalBands, null, true)
                  .addBands(thermalBand, null, true);
    }
    // // A function that scales and masks Landsat 8 (C2) surface reflectance images.
    // function prepSrL8(image) {
    //   // Develop masks for unwanted pixels (fill, cloud, cloud shadow).
    //   var qaMask = image.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
    //   var saturationMask = image.select('QA_RADSAT').eq(0);

    //   // Apply the scaling factors to the appropriate bands.
    //   var getFactorImg = function(factorNames) {
    //     var factorList = image.toDictionary().select(factorNames).values();
    //     return ee.Image.constant(factorList);
    //   };
    //   var scaleImg = getFactorImg([
    //     'REFLECTANCE_MULT_BAND_.,TEMPERATURE_MULT_BAND_ST_B10']);
    //   var offsetImg = getFactorImg([
    //     'REFLECTANCE_ADD_BAND_.,TEMPERATURE_ADD_BAND_ST_B10']);
    //   var scaled = image.select('SR_B.,ST_B10').multiply(scaleImg).add(offsetImg);

    //   // Replace original bands with scaled bands and apply masks.
    //   return image.addBands(scaled, null, true)
    //     .updateMask(qaMask).updateMask(saturationMask);
    // }

    var MovingWindowJoint =function(primary,secondary)
    {

      var windowSize= 1.5 * 24 * 60 * 60 * 1000;
      //Define different timeFilters according modes
      var timeFilter = ee.Filter.maxDifference({
             difference: windowSize,
             leftField: 'system:time_end',
             rightField: 'system:time_start'
            });

      //Define the join
      var saveAllJoin = ee.Join.saveAll({
         matchesKey: 'matchedID',
         ordering: 'system:time_start',
         ascending: true,
         measureKey: 'timeDiff'
      });
      
      // Apply the join.
      var joinedCollection = saveAllJoin.apply(primary, secondary, timeFilter);
      return joinedCollection;
    };

    var combineGAGQ = function(image)
    {
      var TempGQ=ee.List(image.get('matchedID'));
      var imageCollection=ee.ImageCollection(TempGQ);
      image=ee.Image(image).addBands(imageCollection.first());
      return image;
    };

    ////Clip MODIS band by TM Band
    var ClipMODISByTMBoundary =  function(image)
    {
      var TM= image.select(0);
      var MODIS=image.select(1);
      var footprint=TM.get('system:footprint');
      var coordinates =ee.Geometry(footprint).coordinates() ; 
      var boundary=ee.Geometry.Polygon(coordinates);
      MODIS=MODIS.clip(boundary);
      
      //Mask MODIS coloud with TM Mask
      MODIS=MODIS.updateMask(TM.gt(0));
      var FinalImage=ee.Image(TM).addBands(MODIS);
      return FinalImage;
    };
    //////////////////////////////////////////////////////////////////////////////////
    //Main function

    //data preparing
    var  date1 = '2013-01-01';
    var  date2 = '2022-05-30';
    var collectionGQ = ee.ImageCollection('MODIS/006/MOD11A1').filterDate(date1, date2).filterBounds(geometry);
    var GQ = collectionGQ.map(applyScaleFactors_MODISLST).select('LST_Day_1km');
    //print(GQ)

    var collectionTM7 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2').filterDate(date1, date2).filterBounds(geometry).sort('DATE_ACQUIRED').filter(ee.Filter.lte('CLOUD_COVER_LAND',1));
    var TM = collectionTM7.map(applyScaleFactors_TM8LST).select('ST_B10');
    //print(TM) 

    //joint TM with GQ
    var CombinationGAGQ=MovingWindowJoint(TM,GQ);
    //print(CombinationGAGQ);
    //combine to one imagecollection
    var TMGQ=CombinationGAGQ.map(combineGAGQ);
    //print(TMGQ);
    //re-arrange the bands with new name
    TMGQ=ee.ImageCollection(TMGQ).select([0,1],['TMLST','MODISLST']);
    //Clip MODIS band by TM Band
    var TMMODIS=TMGQ.map(ClipMODISByTMBoundary);
    //re-arrange the bands with new name
    TMMODIS=ee.ImageCollection(TMMODIS).select([0,1],['TMLST','MODISLST']);
    print(TMMODIS);

    var LST_PALETTE = {
      palette: [
        '040274', '040281', '0502a3', '0502b8', '0502ce', '0502e6',
        '0602ff', '235cb1', '307ef3', '269db1', '30c8e2', '32d3ef',
        '3be285', '3ff38f', '86e26f', '3ae237', 'b5e22e', 'd6e21f',
        'fff705', 'ffd611', 'ffb613', 'ff8b13', 'ff6e08', 'ff500d',
        'ff0000', 'de0101', 'c21301', 'a71001', '911003'
      ],
      min: 270,
      max: 320,
    };
    //Image index you want to display and output
    var index=2;

    var TM2 = ee.Image(TMMODIS.toList(10).get(index));
    Map.addLayer(TM2.select(0),LST_PALETTE,"TM LST");
    Map.addLayer(TM2.select(1),LST_PALETTE,"MODIS LST");

    // //output to google driver
    // var TMfilename=TM2.get('system:index');
    // print(TMfilename);

    // var footprint=TM2.get('system:footprint');
    // var coordinates =ee.Geometry(footprint).coordinates() ; 
    // var boundary=ee.Geometry.Polygon(coordinates);
    // Export.image(TM2, TMfilename.getInfo(), {
    //   scale: 30, // here MODIS data will be resampled to 30m
    //   region:boundary
    // });
    ```