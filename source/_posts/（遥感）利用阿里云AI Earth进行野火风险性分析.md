---
title: （遥感）利用阿里云AI Earth进行野火风险性分析
date: 2024-03-30 12:44:48
tags:
- 实践
- 踩坑
- 遥感
- Python
- 阿里云
categories:
- 实践
keywords:
- Python
- AI Earth
- 遥感
top_img: https://s2.loli.net/2024/03/30/uxeUYTmFtzHdobc.jpg
cover: https://s2.loli.net/2024/03/30/9mhS3aM8zsnQqXA.jpg
---
`AI Earth` 是一个由阿里云提供支持的遥感算法开发和数据驱动科学平台。利用平台提供的`Python SDK`，我们可以通过编写Python代码来实现对遥感数据的云端处理。平台既提供了在线`Notebook`环境，也支持直接在本地`Jupyter Notebook`编写代码。因为在本地编写更有利于对后续管理，所以本项目也采用了本地`Jupyter Notebook`编写的方式。

# 配置准备

## 安装`AI Earth Engine SDK`

```PowerShell
python -m pip install -U "aie-sdk[engine]"
```

## 注册阿里云账号

[阿里云·云小站](https://www.aliyun.com/minisite/goods?userCode=9k9dfqvv)

## 获取鉴权信息

根据不同的配置环境可以选择不同的[鉴权方式](https://engine-aiearth.aliyun.com/docs/page/api?d=07f36f#heading-22:~:text=%E7%BC%96%E5%86%99%E4%BB%A3%E7%A0%81-,%E9%89%B4%E6%9D%83%E5%8F%8A%E5%88%9D%E5%A7%8B%E5%8C%96,-%E6%82%A8%E5%8F%AF%E4%BB%A5%E5%9F%BA%E4%BA%8E)
- 使用在线`NoteBook`环境：无须鉴权
- 使用本地模式，但是不需要长期有效：使用token鉴权
- 使用本地模式，但是希望鉴权长期有效：使用阿里云 AccessKey 鉴权
- 使用本地模式，但是希望使用Aliyun STS 鉴权

[在AI Earth生成token](https://engine-aiearth.aliyun.com/#/utility/auth-token)

# 编写代码

## 鉴权

选取对应的方式进行鉴权

```Python
import aie

# 在线NoteBook不需要填入参数
aie.Authenticate()
# 使用token进行鉴权
aie.Authenticate(token='token')
# 使用 aliyun access_key_id进行鉴权
aie.Authenticate(access_key_id="*", access_key_secret="*")
# 使用 aliyun RAM STS 进行鉴权(需要 aiearth-core>=1.0.3)
aie.Authenticate(access_key_id='*', access_key_secret='*', access_key_security_token='*')

# 初始化
aie.Initialize()
```

## 确定研究区域

分别提取广西和广东的矢量边界并进行合并，然后在地图上表示出来。

```Python
feature_collection_1 = aie.FeatureCollection('China_Province') \
                        .filter(aie.Filter.eq('province', '广西壮族自治区'))

feature_collection_2 = aie.FeatureCollection('China_Province') \
                        .filter(aie.Filter.eq('province', '广东省'))

union_fc = feature_collection_1.merge(feature_collection_2).union()


union_geometry = union_fc.geometry()


map = aie.Map(
    center=union_fc.getCenter(),
    height=800,
    zoom=6
)

vis_params = {
    'color': '#00FF00'
}

map.addLayer(
    union_fc,
    vis_params,
    'region',
    bounds=union_fc.getBounds()
)

map

# 将边界导出到持久化数据空间中
# aie.Export.feature.toAsset(union_fc, "两广地区边界").start()
```
AI Earth提供了`map`方法用以渲染地图组件。将项目流程切分为NoteBook的多个cell可以让组件进行独立渲染，这将更方便我们观察每个步骤的结果。后续的步骤也将采用这样的方式。

## 获取地物分类

使用的数据集： https://engine-aiearth.aliyun.com/#/dataset/ESA_WORLD_COVER_V200

```Python
# 调用数据集并筛选包含研究区域的部分
dataset_classify = aie.ImageCollection('ESA_WORLD_COVER_V200') \
             .filterBounds(union_geometry) 

img_cover = dataset_classify.mosaic() \
                            .clip(union_geometry) \
                            .rename(["cover"])

# 后续的小节不会再展示map的配置
map = aie.Map(
    center=union_fc.getCenter(),
    height=800,
    zoom=6
)
vis_params = {
    'SR_Bands': 'cover',
    'min': 10.0,
    'max': 100.0,
    'palette': [
        '#006400','#ffbb22','#ffff4c','#f096ff',
        '#fa0000','#b4b4b4','#f0f0f0','#0064c8',
        '#0096a0','#00cf75','#fae6a0'
    ]
}

map.addLayer(
    img_cover,
    vis_params,
    'cover',
    bounds=union_fc.getBounds()
)
map                
```

栅格数据的处理方式大同小异，故下文只展示各种数据需要特别处理的地方。

## 获取历史火灾数据集

使用的数据集： https://engine-aiearth.aliyun.com/#/dataset/MODIS_MCD64A1_006

```Python

# 选择近十年来发生过火灾的区域
dataset_firepoint = aie.ImageCollection('MODIS_MCD64A1_006') 
                        .filterDate('2012-01-01', '2022-12-31')


# 用or运算提取所有火点然
# 用mask来为nodata填0
# 裁切对象并重命名
img_firepoint = dataset_firepoint.select(['BurnDate']).Or() \
                            .mask() \
                            .clip(union_geometry) \
                            .rename(["firepoint"])
```

## 计算坡度坡向

使用的数据集： https://engine-aiearth.aliyun.com/#/dataset/Copernicus_DEM_30M

```Python
# 计算坡度和坡向
img_slope = aie.Terrain.slope(img_DEM).rename(["slope"])
img_aspect = aie.Terrain.aspect(img_DEM).rename(["aspect"])
```

## 生物气候变量

使用的数据集： https://engine-aiearth.aliyun.com/#/dataset/ERA5_LAND_MONTHLY

该数据集的各个波段有不同的含义，我们只需提取要用的内容

```Python
# 平均2m温度             
dataset_avg_temp = dataset_Bioclimatic.select(['temperature_2m'])
img_avg_temp = dataset_avg_temp.mean().clip(union_geometry).rename(["avg_temp"])
# 平均地表温度
dataset_skin_avg_temp = dataset_Bioclimatic.select(['skin_temperature'])
img_skin_avg_temp = dataset_skin_avg_temp.mean().clip(union_geometry).rename(["skin_avg_temp"])
# 平均气压
dataset_surface_pressure = dataset_Bioclimatic.select(['surface_pressure'])
img_surface_pressure = dataset_surface_pressure.mean().clip(union_geometry).rename(["surface_pressure"])
# 年降水量
dataset_precipitation = dataset_Bioclimatic.select(['total_precipitation'])
img_precipitation = dataset_precipitation.sum().clip(union_geometry).rename(["precipitation"])
# 年蒸发量
dataset_evaporation = dataset_Bioclimatic.select(['total_evaporation'])
img_evaporation = dataset_evaporation.sum().clip(union_geometry).rename(["evaporation"])

```

## WET

利用Landsat9的数据计算湿度指标

使用的数据集：https://engine-aiearth.aliyun.com/#/dataset/LANDSAT_LC09_C02_T1_L1

```Python
dataset = aie.ImageCollection('LANDSAT_LC09_C02_T1_L1') \
             .filterBounds(union_fc) \
             .filterDate('2021-01-01', '2022-12-31') \
             .filter(aie.Filter.lte('eo:cloud_cover', 30.0))

# 去云函数
def removeLandsatCloud(image):
    # cloudShadowBitMask = (1 << 4)
    cloudsBitMask = (1 << 3)
    qa = image.select('QA_PIXEL')
    mask = qa.bitwiseAnd(aie.Image(cloudsBitMask)).eq(aie.Image(0))
    return image.updateMask(mask)
    
images_no_cloud = dataset.map(removeLandsatCloud)

image_L9C2L2 = images_no_cloud.mosaic().clip(union_geometry)

# 对于Landsat9
# WET = 0.1511 B1 + 0.1973 B2 + 0.3283 B3 + 0.3407 B4 - 0.7171 B5 - 0.4559 B6
img_wet = image_L9C2L2.select('B2').multiply(aie.Image(0.1511)) \
    .add(image_L9C2L2.select('B3').multiply(aie.Image(0.1973))) \
    .add(image_L9C2L2.select('B4').multiply(aie.Image(0.3283))) \
    .add(image_L9C2L2.select('B5').multiply(aie.Image(0.3407))) \
    .add(image_L9C2L2.select('B6').multiply(aie.Image(0.7171))) \
    .add(image_L9C2L2.select('B7').multiply(aie.Image(0.4559))) \
    .rename(["wet"])
```
## 土壤分类

使用的数据集： https://engine-aiearth.aliyun.com/#/dataset/OPENLANDMAP_SOL_SOL_TEXTURE-CLASS_USDA-TT_M_V02

过程：略

## NDVI

使用的数据集： https://engine-aiearth.aliyun.com/#/dataset/MODIS_MOD13Q1_006
过程：略

# 使用自适应分类器

## 划分数据

```Python
img_all = img_firepoint.addBands(img_aspect) \
                        .addBands(img_slope) \
                        .addBands(img_DEM)  \
                        .addBands(img_precipitation) \
                        .addBands(img_cover) \
                        .addBands(img_soil) \
                        .addBands(img_surface_pressure) \
                        .addBands(img_avg_temp) \
                        .addBands(img_ndvi) \
                        .addBands(img_evaporation) \
                        .addBands(img_wet) \
                        .clip(union_geometry)
                        
# 从指定范围内取样
samples = img_all.sampleRegion(union_geometry, 800, True)

# 在训练样本中增加一列随机数, 选取80%的样本为训练样本, 选取20%的样本为验证样本
sample = samples.randomColumn()
training_sample = sample.filter(aie.Filter.lte('random', 0.80))
validation_sample = sample.filter(aie.Filter.gt('random', 0.80))
```


## 使用自适应分类器
```Python
# 创建自适应集成分类器，并进行训练
trained_classifier = aie.Classifier.adaBoost(10)
trained_classifier = trained_classifier.train(training_sample, 
                                                "firpoint", 
                                                img_all.bandNames().getInfo())

# 获取训练样本的混淆矩阵, 并对训练精度进行评估
train_accuracy = trained_classifier.confusionMatrix()
print('Training error matrix:', train_accuracy.getInfo())
print('Training overall accuracy:', train_accuracy.accuracy().getInfo())


# 使用验证集对分类器进行评估
validation = validation_sample.classify(trained_classifier)
validation_accuracy = validation.errorMatrix(label, 'classification')
print('Validation error matrix:', validation_accuracy.getInfo())
print('Validation accuracy:', validation_accuracy.accuracy().getInfo())

# 使用训练好的分类器对影像进行分类
img_classified = img_all.classify(trained_classifier)
```
## 查看结果

```Python
print(img_classified.getInfo())


map = aie.Map(
    center=union_fc.getCenter(),
    height=800,
    zoom=6
)
map.addLayer(
    img_classified,
    {
        'min': 0,
        'max': 1,
        'palette': ["0,255,255","255,0,0"]
    },
    'aie_classification',
    bounds=union_fc.getBounds()
)
map
```

# 辅助函数

为了节省调试实践，我在做这个项目的时候封装了一些函数
```Python
from math import sqrt

# 渲染地图前自动获取栅格数据的最大值和最小值
def get_max_and_min(img_target:aie.Image,region:aie.Geometry=union_geometry):
    reducer = aie.Reducer.max().combine(reducer2=aie.Reducer.min(), sharedInputs=True)
    result = img_target.reduceRegion(reducer, region)
    max,min = list(result.getInfo().values())
    return max,min

# 用平均值来填充nodata
def fix_with_mean(img_target:aie.Image,region:aie.Geometry=union_geometry):
    reducer = aie.Reducer.mean()
    result = img_target.reduceRegion(reducer, union_geometry)

    mean_tag = list(result.getInfo().keys())[0]
    target_mean = result.getInfo()[mean_tag] 
    
    img_mean = aie.Image(target_mean).clip(union_geometry)
    img_target = img_target.unmask(img_mean)
    img_target = img_target.updateMask(img_target.mask())
    
    return img_target

# 将数据导出到持久化空间
def save(img_target:aie.Image, filename:str=None):
    if filename is None:
        filename = img_target.bandNames().getInfo()[1]

    # 如果不按原分辨率导出的话会有明显的裂缝
    # 所以在不知道分辨率的情况下要手动计算
    reducer = aie.Reducer.min()
    result = img_target.pixelArea().reduceRegion(reducer, union_geometry)
    area = sqrt(result.getInfo()['area_min'])
    
    name = img_target.bandNames().getInfo()[0]
    print(name + ":" + str(area))

    aie.Export.image.toAsset(img_target, filename, area).start()
```



# 相关链接：

阿里云AI Earth： https://engine-aiearth.aliyun.com/#/

利用AI Earth给Landsat9数据去云： https://engine-aiearth.aliyun.com/docs/page/case?d=6cf4e8

WET湿度指数计算： http://www.yndxxb.ynu.edu.cn/yndxxbzrkxb/article/doi/10.7540/j.ynu.20190174?viewType=HTML