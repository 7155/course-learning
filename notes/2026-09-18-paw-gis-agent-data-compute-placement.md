# 2026-09-18｜PAW GIS Agent：本地 GIS、GEE 与数据放置策略

> 这一篇只解决一个核心问题：**Agent 如何决定“在哪里算、数据要不要搬、结果要不要拿回来”。**
>
> 目标不是记 GEE API，而是从真实 GIS 任务重新推导 Local GIS、GEE、Asset、Tile、Evaluate、Export 为什么存在，以及 PAW 的 GIS Agent 应该如何编排它们。

---

# 0. 最终只保留这一张总图

~~~text
                         User Task
                            │
                            ▼
                       PAW Agent
                 reasoning / planning
                            │
               ┌────────────┼────────────┐
               │            │            │
               ▼            ▼            ▼
          Local GIS        GEE         Map/View
          GeoPandas      Earth Engine   Leaflet
          Rasterio          Cloud          │
          Shapely                         UI
               │            │            │
               └──── result / state ─────┘
                            │
                            ▼
                       Agent Loop
~~~

一句话：

> **Agent 负责决定“数据放哪、计算放哪、结果是否搬回来”；Local GIS 和 GEE 是两个执行后端，右侧地图只是交互与显示层。**

---

# 1. 从一个本地 Shape 开始

假设本地有：

~~~text
data/buildings.shp
~~~

用户说：

> 把建筑外扩 500 米。

最合理的执行链：

~~~text
buildings.shp
     ↓
Local GeoPandas / Shapely
     ↓
reproject
     ↓
buffer 500m
     ↓
buffer.geojson
~~~

这里没必要碰 GEE，因为：

- 数据已经在本地；
- 只是简单工程几何；
- 本地算子可以完成；
- 上传云端只会增加网络和状态管理成本。

所以第一条原则：

> **数据已经在本地、且本地算子能低成本完成时，优先 Local GIS。**

---

# 2. 为什么又需要 GEE？

把任务改成：

> 分析这些建筑周围过去 5 年 Sentinel-2 的植被变化。

本地可能只有几十 KB 的 Shape，但遥感数据可能是几十 GB、几百 GB甚至更多。

错误思路：

~~~text
GEE 大量影像
    ↓
全部下载到 Local
    ↓
本地再算
~~~

更合理：

~~~text
Local Shape
    ↓
发送小型 geometry
    ↓
GEE
    ↓
Sentinel-2 / NDVI / time series
    ↓
只返回需要的统计
~~~

这就是：

> **move computation to data，而不是 move huge data to computation。**

---

# 3. 本地 Shape 到 GEE，有两种方式

## 3.1 小数据、一次性使用：Inline Geometry

例如只有：

~~~text
10 个候选地块
20 个训练样本
1 个 AOI
~~~

Agent 可以：

~~~text
Local GeoJSON
    ↓
转换 EPSG:4326
    ↓
ee.Geometry / ee.FeatureCollection
    ↓
随请求提交给 GEE
~~~

适合：

- 少量候选地块；
- 用户临时框选 AOI；
- 少量训练样本；
- 一次性分析范围。

不需要建立永久云端资产。

## 3.2 大数据、反复使用：GEE Asset

如果数据变成：

~~~text
几十万宗地
全国道路
固定保护区
长期维护的训练样本
~~~

每次都从本地重新传输就很浪费。

于是：

~~~text
Local Shapefile / GeoJSON
          ↓
      upload once
          ↓
       GEE Asset
          ↓
projects/.../assets/xxx
          ↓
后续脚本直接引用 Asset ID
~~~

所以 Asset 的本质不是“云端算得更高级”，而是：

> **给需要反复参与云计算的数据一个持久化云端位置。**

---

# 4. GEE 算完以后，也不一定要下载

假设 GEE 已经生成 slope、classification、NDVI 等影像。

Agent 下一步不是机械地“下载结果”，而是判断：

> 用户后面到底要“看”、要“几个数字”，还是要“真正拿文件继续处理”？

于是出现三条结果通路。

---

# 5. 只需要看：Tile

~~~text
GEE Image
   ↓
Map Tile URL
   ↓
Leaflet
   ↓
右侧显示
~~~

关键区别：

~~~text
地图已经显示
≠
GeoTIFF 已经下载
~~~

Tile 只是可视化。

---

# 6. 只需要数字或小矢量：Evaluate

例如选址只需要：

~~~text
candidate_A
slope_p90 = 6.8°
water_fraction = 0.01
forest_fraction = 0.12
~~~

最合理：

~~~text
GEE Image
   ↓
reduceRegion / reduceRegions
   ↓
FeatureCollection / Dictionary
   ↓
evaluate
   ↓
small JSON
   ↓
Local
~~~

本地随后可以：

~~~text
云端指标
+
本地面积
+
本地禁区结果
    ↓
attribute join
    ↓
最终筛选
~~~

因此：

> **只需要派生指标时，不应该为了几个数字下载整幅 Raster。**

---

# 7. 后面必须本地栅格处理：Export

例如：

> GEE 完成分类后，在本地 Rasterio 中 polygonize，再与道路做拓扑分析。

此时 Tile 和 Evaluate 都不够，必须：

~~~text
GEE Classification Image
          ↓
      Export Task
          ↓
Cloud Storage / Drive / Asset
          ↓
      task status
          ↓
       GeoTIFF
          ↓
         Local
          ↓
       Rasterio
          ↓
      polygonize
          ↓
      GeoPandas
~~~

所以：

> **只有下游明确需要本地 Raster 时，Agent 才应该真正把大影像搬回来。**

---

# 8. Agent 真正做的是 Data + Compute Placement

GIS Agent 不是简单地：

~~~text
识别用户意图
↓
选一个 GIS Tool
~~~

而应该是：

~~~text
用户任务
   ↓
数据在哪？
   ↓
数据多大？
   ↓
是否反复使用？
   ↓
哪个执行平面更合适？
   ↓
下游还需要什么？
   ↓
决定数据是否搬迁
   ↓
决定结果返回形式
~~~

压缩成：

~~~text
只看                 → Tile
只要数字             → Evaluate
只要小型矢量         → GeoJSON / FeatureCollection
需要本地 Raster 运算 → Export GeoTIFF
小 Shape 一次使用    → Inline Geometry
大 Shape 反复使用    → GEE Asset
简单本地几何         → Local GIS
大规模遥感计算       → GEE
~~~

真正的判断变量是：

~~~text
数据位置
+
数据规模
+
计算规模
+
复用频率
+
下游位置
+
网络搬运成本
+
API 限制
+
CRS / scale
+
权限 / 隐私
~~~

---

# 9. 两个反例理解 Data Locality

## 反例 A：GEE 有 100 GB 影像，本地只有 50 KB Polygon

错误：

~~~text
100 GB Sentinel
    ↓
下载本地
~~~

正确：

~~~text
50 KB Polygon
    ↓
GEE
    ↓
云端计算
    ↓
返回统计
~~~

## 反例 B：本地有 80 GB 私有航测影像，GEE 只提供小型约束结果

未必应该把 80 GB 全部上传。

可能更合理：

~~~text
GEE
 ↓
生成水域/土地覆盖约束
 ↓
下载小型 GeoJSON / Raster
 ↓
Local 80 GB imagery
 ↓
本地继续计算
~~~

所以不能简单总结为“云端数据云端算，本地数据本地算”。

---

# 10. PAW Earth Agent 当前真实边界

当前已经具备：

~~~text
Agent Loop                         ✅
Local GIS                         ✅
小型 Geometry → GEE               ✅
GEE 云端计算                       ✅
GEE → small statistics            ✅
GEE → FeatureCollection            ✅
GEE → map tiles                    ✅
Agent → Map structured command     ✅
Map selection → Agent context      ✅
~~~

仍需补齐：

~~~text
Local Shapefile
      ↓
automatic GEE Asset upload         ⚠️

GEE Image
      ↓
Export Task
      ↓
status polling
      ↓
download GeoTIFF
      ↓
Local Rasterio                     ⚠️
~~~

所以当前准确说法：

> **小数据的本地 ↔ 云端闭环已经打通；大数据的自动 Asset 上传和 Export 下载仍然需要完整的数据桥。**

---

# 11. 下一步不是只加工具，而是加 Data Planner

目标架构：

~~~text
                       Agent
                         │
                         ▼
                Data / Compute Planner
                         │
              ┌──────────┴──────────┐
              │                     │
        Local Execution        GEE Execution
              │                     │
              └──────────┬──────────┘
                         │
                     Data Mover
             ┌───────────┼────────────┐
             │           │            │
      inline geometry   Asset      Export
             │           │            │
             └───────────┴────────────┘
                         │
                    Artifact Store
~~~

Data Planner 的输出最好显式记录：

~~~json
{
  "executionPlane": "gee",
  "inputPlacement": "inline_geometry",
  "resultMode": "evaluate",
  "reason": "small vector input + large remote imagery + only statistics required"
}
~~~

这样 Agent 的决策才能被检查、复现、评测和 Debug。

---

# 12. Artifact 必须成为一等对象

不要只存 result.tif 或 result.geojson。

应该记录：

~~~json
{
  "artifactId": "art_xxx",
  "source": "gee",
  "dataset": "ESA/WorldCover/v200",
  "crs": "EPSG:4326",
  "scale": 10,
  "runId": "run_xxx",
  "parentArtifacts": ["art_input_xxx"],
  "localPath": "pred_results/result.tif"
}
~~~

真正的 GIS Agent 必须回答：

- 这个结果从哪里来的？
- 哪一次运行产生的？
- CRS 和 scale 是什么？
- 是 Tile、统计、矢量还是完整 Raster？
- 是否真的已经下载到本地？

---

# 13. 这部分天然适合做 Eval

## Eval 1：CRS

输入：

~~~text
EPSG:4326 polygon
buffer 500m
~~~

检查 Agent 是否先重投影，而不是直接 buffer 500 degree。

## Eval 2：Data Placement

输入：

~~~text
本地 20 KB Polygon
+
GEE 多年 Sentinel
+
只需要平均 NDVI
~~~

检查 Agent 是否把 Polygon 发给 GEE，而不是下载所有影像。

## Eval 3：Result Mode

GEE 图层显示成功后检查：

> Agent 是否把“Tile 已显示”错误描述成“GeoTIFF 已下载”。

## Eval 4：Failure / Provenance

制造 GEE 失败，检查：

~~~text
失败是否进入 Observation
Agent 是否错误宣称完成
runId 是否保留
旧结果是否被误当成新结果
~~~

---

# 14. 电网选址 Demo 如何体现整套能力

~~~text
Local candidates.geojson
Local exclusions.geojson
        │
        ▼
Local inspect / reproject
        │
        ▼
buffer + difference
        │
        ▼
small candidate geometry
        │
        ▼
GEE DEM / WorldCover / Water
        │
        ▼
slope / forest / water metrics
        │
        ▼
small FeatureCollection / JSON
        │
        ▼
Local attribute join
        │
        ▼
local route planning
        │
        ▼
GeoJSON LineString
        │
        ▼
Map visualization
~~~

如果后续需要本地处理分类 Raster，再追加：

~~~text
GEE
 ↓
Export
 ↓
GeoTIFF
 ↓
Rasterio
~~~

---

# 15. 面试压缩版

## 一句话

> **PAW GIS Agent 本质上是一个混合 GIS workflow orchestrator：Agent 根据数据位置、规模、复用频率和下游需求，动态选择 Local GIS 或 GEE，并决定使用 Inline Geometry、Asset、Evaluate、Tile 或 Export 来完成数据跨执行平面的流动。**

## 30 秒版本

> 本地 GeoPandas、Shapely、Rasterio 负责工程几何和本地文件计算，GEE 负责大规模遥感数据。Agent 不只是选算子，还要做 data locality 和 compute locality 决策。比如一个几十 KB 的候选地块和几百 GB 的 Sentinel 数据做分析时，不会把影像下载下来，而是把小几何送到 GEE，只返回坡度或 NDVI 统计。如果数据很大且需要长期复用，就上传 GEE Asset；如果云端结果后续必须 Rasterio 处理，才通过 Export 把 GeoTIFF 拿回本地。

## 深挖追问

1. 为什么 Tile 显示成功不等于 GeoTIFF 已经下载？
2. Inline Geometry 与 GEE Asset 的边界是什么？
3. 什么情况下应该把云端结果拿回本地？
4. Agent 如何避免为了几个统计值下载整幅 Raster？
5. Data Planner 应该记录哪些决策信息？
6. 如何评测 Agent 的数据放置决策是否正确？
7. GEE Export 为什么需要独立的 Task 状态管理？

---

# 16. 最终心智模型

~~~text
Agent = 决策与编排

Local GIS
= 本地文件 / 工程几何 / 本地 Raster

GEE
= 大规模云端遥感计算

Inline Geometry
= 小数据临时送云端

Asset
= 大数据 / 高频复用的云端持久化

Tile
= 看

Evaluate
= 拿小结果

Export
= 真正搬大结果

Artifact / Run
= 证明“算了什么、在哪里算、结果从哪来”
~~~

最后只记一句：

> **GIS Agent 的高级之处不是“工具多”，而是它能正确决定 computation 和 data 应该在哪里，并且只搬真正需要搬的数据。**
