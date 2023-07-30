---
title: '利用Sentinel卫星影像数据研究某市地震引发的地表形变'
date: 2023-07-30 18:50:23
update: 
tags:
- 实践
- ENVI
- 遥感
categories:
- 实践
keywords: 
- ENVI
highlight_shrink: false
top_img: https://s2.loli.net/2023/07/30/EeQU3pMGNI5dxsc.jpg
cover: https://s2.loli.net/2023/07/30/QAEfBgnHG4zU9l6.jpg
---
# 0、 前情概要

据中国地震台网正式测定：07月03日20时57分某市发生3.7级地震，震源深度9千米。某分析小组将用遥感卫星影像数据分析地震前后地表发生的形变。

# 1、 检查 SARscape 在 ENVI 中的配置

1. 安装ENVI 以及 SARscape扩展
2. 下载研究地区对应时段的Sentinel卫星影像和DEM
3. 检查 SARscape 在 ENVI 中的配置
4. 有条件的话还可以准备GPC文件，但它不是必须的

![pre_common](https://s2.loli.net/2023/07/30/ERYpJxrebGSFH7u.png)

设置内存占用限制，检查OpenCL硬件加速的配置情况， ~~我这里没有独显就只开了核显~~ 。

![](https://s2.loli.net/2023/07/30/m6bI1uSksQCvGHN.png)

应用ENVI自带的Sentinel参数设置

# 2、 加载 Sentinel 卫星影像

![](https://s2.loli.net/2023/07/30/95lYTSQ3jawsUNg.png)

依次导入所有影像，选择统一输出的目录，点击 `Exec` 开始读取。

![](https://s2.loli.net/2023/07/30/F3clL2kHJCK5u4p.png)

（SARscape 默认为每个步骤的输出文件重命名为 `原文件名` 加 `特定后缀` ，有特殊需求可以在 `Parameters` 里更改。）

# 3、 基线估算

打开基线估算
> /SARscape/Interferometry/Interferometric Tools/Baseline Estimation

![](https://s2.loli.net/2023/07/30/XkhLMgW8SQ6vdHB.png)

输入主从影像的 `_list` 文件，点击 `Exec` 开始执行。

```log
Normal Baseline (m) = -40.038	Critical Baseline min - max(m) = [-6475.240] - [6475.240]
Range Shift (pixels)    = 16.016
Azimuth Shift (pixels)  = -13414.929
Slant Range Distance (m)  = 880579.036
Absolute Time Baseline (Days) = 12
Doppler Centroid diff. (Hz) = 4901.873	Critical min-max (Hz) = [-486.486] - [486.486]
2 PI Ambiguity height (InSAR) (m) = 390.992
2 PI Ambiguity displacement (DInSAR) (m) = 0.028
1 Pixel Shift Ambiguity height (Stereo Radargrammetry) (m) = 32843.320
1 Pixel Shift Ambiguity displacement (Amplitude Tracking) (m) = 2.330
Master Incidence Angle = 39.869	Absolute Incidence Angle difference = 0.003
Pair potentially suited for Interferometry, check the precision plot

```

基线估算的结果显示，这两景数据的空间基线为`-40.038`米，位于临界基线 `±6475.240` 米之内，时间基线 `12` 天，做DInSAR的一个相位变化周期代表的地形变化为 `0.028` 米。


# 4、 DInSAR Displacement WorkFlow 工作流

打开 DInSAR Displacement WorkFlow 工作流。

> /SARscape/Interferometry/DInSAR Displacement Workflow

![](https://s2.loli.net/2023/07/30/EoUhy6XV35TSlem.png)

选择主从影像，引用 DEM 。 *（没有准备DEM可以在 Tolbox/SARscape/General Tools/DEM Extraction/SRTM-3 Version 4 里下载 ）*

![](https://s2.loli.net/2023/07/30/wGUaZ5ne7vdLt6l.png)

**主要不要点到 `Next` 去了**，这样做会很拖时间！

左侧的选项栏可以直接调整每个步骤的参数，调完所有的之后再启动可以无人值守跑完整个流程。

![](https://s2.loli.net/2023/07/30/DjhSVgFCvHriZAX.png)

如果你的影像分辨率很高，就在左侧的选项栏找到`Adaptive Filter and Coherenace Generation`使用对高分辨率影像更友好算法。（分辨率不到5米的可以跳过这步）

有准备的数据有控制点文件可以在这里导入，没有控制点对这次试验这次实验影响也不大。

如果没有什么其他特殊需求的话其他的地方，其他参数就都按自动填写的默认值就好了。

![](https://s2.loli.net/2023/07/30/FxG9Jz7ApNBZhra.png)

都调完了后就在`Output`选择结果文件额的输出路径。

然后回到`Input`直接点 `NEXT >>>` 开始任务。~~接下来就是在去菜市路上想想今晚该吃什么了~~

我的高精度影像数据在工作站满载跑了一个晚上才跑完，所以配置比较差的朋友更要主要合理安排时间。

> 附配置单
> 
> E5 2666v3 * 2
> 
> 1080 Ti * 1
> 
> 96 GB + 1TB

# 5、 结果分析

万幸的是，这一晚上都没有弹出报错。第二天到办公室就刚好看见了完成的结果图。图中可以看到，颜色越红的区域就是影响越大的区域。

![](https://s2.loli.net/2023/07/30/FxG9Jz7ApNBZhra.png)

# 6、 GIS出图*

（这一步不是必须的）

把 ENVI 的结果图转成通用格式放进GIS类软件（如ArcGIS）加工好就可以制成专题图了。