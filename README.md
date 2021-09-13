[TOC]

# GIS常用算法

作为一个 GISer，在日常 WebGIS 开发中，会常用到的 [turf.js](http://turfjs.org/)，这是一个地理空间分析的 JavaScript  库，经常搭配各种 WebGIS JsAPI 使用，如 [Leaflet](https://leafletjs.com/)、[MapboxGL](https://www.mapbox.com/mapbox-gljs)、[OpenLayers](https://openlayers.org/) 等；在后台 Java 开发中，也有个比较强大的GIS库，[GeoTools](https://www.geotools.org/)，里面包含构建一个完整的地理信息系统所需要的全部工具类；数据库端常用是 [postgis](https://postgis.net/) 扩展，需要在 [PostgreSQL](https://www.postgresql.org/) 数据库中引入使用。

然而在开发某一些业务系统的时候，有些需求只需要调用某一个 GIS 算法，简单的几行代码即可完成，没有必要去引用一个 GIS 类库。

而且有些算法在这些常用的GIS类库中没有对应接口，就比如在下文记录的这几种常用算法中，求垂足、判断线和面的关系，在 [turf.js](http://turfjs.org/) 就没有对应接口。

下面文章中是我总结的一些常用 GIS 算法，这里统一用JavaScript 语言实现，因为 JS 代码相对比较简洁，方便理解其中算法逻辑，也方便在浏览器下预览效果。

在具体应用时可以根据具体需求，翻译成 Rust、C/C++、Java、C#.NET、Python 等语言来使用。

文中代码大部分为之前遇到需求时在网上搜索得到，然后自己根据具体需要做了优化修改，通过这篇文章做个总结收集，也方便后续使用时查找。


## 1 算法介绍与示例代码

以下方法中传参的点、线、面都是对应 GeoJSON 格式中 `coordinates`，方便统一调用。

GeoJSON 标准参考：https://geojson.org/

### 简单的类型约定

``` js
/**
 * @typedef {[number, number]} LonLat
 * @typedef {LonLat[]} LineString
 * @typedef {LonLat[]} Ring
 * @typedef {LonLat[][]} Polygon
 * @typedef {[LonLat, LonLat]} LineSegment
 */
```

以下所有算法函数用到三个简单的输入类型，两个数构成的数组称为 `LonLat`，即经纬度点；`LonLat` 数组，即一系列的经纬度点，称为折线，特殊情况下，两个点构成的叫线段 `LineSegment`；多个 `LonLat` 点除了表示折线外，还可以表示闭合的单一多边形 `Ring`，但是在 GeoJSON 中， `Polygon` 是允许有多个简单多边形构成的，即 `LonLat[][]`。

``` js
// LonLat 经纬度点
const p1 = [112.5, 41.4]

// LineString 折线
const l1 = [[112.5, 41.4], [112.5, 41.5], [112.5, 41.6]]

// LineSegment 线段
const seg = [[112.5, 41.4], [112.5, 41.5]]

// Ring 单环
const ring = [[112.5, 41.4], [112.9, 41.5], [113.1, 40.6], [112.5, 41.4]]

// Polygon 多边形
const polygon = [[[112.5, 41.4], [112.9, 41.5], [113.1, 40.6], [112.5, 41.4]]]
```



### 1.1 经纬度点之间的距离 getDistance

> 适用场景：测量

```js
const p1 = [112.5, 41.2]
const p2 = [121.3, 22.4]
const distance = getDistance(p1, p2) // 2249255.8
```



### 1.2 根据已知线段以及到起点距离，求目标点坐标 getLinePoint

> 适用场景：封闭管段定位问题点

```js
const l1 = [[116.01,40.01],[116.52,40.01]]
const offset = 8500
getLinePoint(l1, offset) // return [116.1096913821572, 40.01]
```



### 1.3 已知点、线段，求垂足 getFootPoint

垂足可能在线段上，也可能在线段延长线上。

> 适用场景：求垂足

```js
const l1 = [[116.01,40.01],[116.52,40.01]]
const p1 = [116.35, 40.08]
getFootPoint(l1, p1) // return [116.35, 40.01]
```



### 1.4 线段上距离目标点最近的点 getShortestPointInLine

不同于上面求垂足方法，该方法求出的点肯定在线段上。

如果垂足在线段上，则最近的点就是垂足，如果垂足在线段延长线上，则最近的点就是线段某一个端点。

> 适用场景：根据求出最近的点计算点到线段的最短距离

```js
const l = [[116.01,40.01],[116.52,40.01]]
const p = [116.72, 40.18]
getShortestPointInLine(l, p) // return [116.52, 40.01]
```



### 1.5 点缓冲 bufferPoint

这里缓冲属于测地线方法，由于这里并没有严格的投影转换体系，所以与标准的测地线缓冲还有些许误差，不过经测试，半径 100km 内，误差基本可以忽略。

具体缓冲类型可看下之前的文章 [你真的会用PostGIS中的buffer缓冲吗？](https://blog.csdn.net/gisarmory/article/details/109646712)

> 适用场景：根据点和半径画圆

```js
const p = [116.72, 40.18]
bufferPoint(p, 50000, 64) // 返回一个数组，有 64 个 LonLat
```



### 1.6 点和面关系 pointInPolygon

该方法采用射线法思路实现。

了解射线法可参考：https://blog.csdn.net/qq_27161673/article/details/52973866

这里已经考虑到环状多边形的情况。

> 适用场景：判断点是否在面内

```js
const polygon = [
  [
    [116.1, 39.5],
    [116.1, 40.5],
    [116.9, 40.5],
    [116.9, 39.5],
  ],
  [
    [116.3, 39.7],
    [116.3, 40.3],
    [116.7, 40.3],
    [116.7, 39.7],
  ], 
  [
    [116.4, 39.8],
    [116.4, 40.2],
    [116.6, 40.2],
    [116.6, 39.8],
  ]
]

pointInPolygon([116.35, 40.08], polygon) // return 0
pointInPolygon([116.35, 40.08], polygon) // return 1
pointInPolygon([116.35, 40.08], polygon) // return 2
```



### 1.7 线段与线段的关系 intersectLineAndLine

> 适用场景：判断线和线的关系

```js
const l1 = [[116.01,40.01],[116.52,40.01]]
const l2 = [[116.33,40.21],[116.36,39.76]]
intersectLineAndLine(l1, l2) // return 1
```



### 1.8 线和面关系 intersectLineAndPolygon & intersectLineAndRing

> 适用场景：判断线与面的关系

该方法考虑到环状多边形的情况，且把相切情况分为了内切和外切。

参考链接：https://www.cnblogs.com/xiaozhi_5638/p/4165353.html

```js
const l1 = [[116.01,40.01],[116.52,40.01]]
const l2 = [[116.33,40.21],[116.36,39.76]]
const l3 = [[116.3,39.7],[116.4,40.1]]
const polygon = [
  [
    [116.1, 39.5],
    [116.1, 40.5],
    [116.9, 40.5],
    [116.9, 39.5],
  ],
  [
    [116.3, 39.7],
    [116.3, 40.3],
    [116.7, 40.3],
    [116.7, 39.7],
  ],
  [
    [116.4, 39.8],
    [116.4, 40.2],
    [116.6, 40.2],
    [116.6, 39.8],
  ]
]

intersectLineAndPolygon(l1, polygon) // return 1
intersectLineAndPolygon(l2, polygon) // return 0
intersectLineAndPolygon(l3, polygon) // return 4
```



### 1.9 GeoJSON 面转线 convertPolygonToPolyline

> 适用场景：只有 GeoJSON 面数据，获取线的边界

```js
const polygon = {
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {},
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [
              112.34344482421875,
              22.172144917381736
            ],
            [
              114.74395751953125,
              22.172144917381736
            ],
            [
              113.51348876953124,
              23.274150156028483
            ],
            [
              112.34344482421875,
              22.172144917381736
            ]
          ]
        ]
      }
    }
  ]
}
convertPolygonToPolyline(polygon)

// 输出
/*
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {},
      "geometry": {
        "type": "MultiLineString",
        "coordinates": [
          [
            [
              112.34344482421875,
              22.172144917381736
            ],
            [
              114.74395751953125,
              22.172144917381736
            ],
            [
              113.51348876953124,
              23.274150156028483
            ],
            [
              112.34344482421875,
              22.172144917381736
            ]
          ]
        ]
      }
    }
  ]
}
*/
```



## 2 在线示例

在线示例：[GISAlgorithm Demo](http://gisarmory.xyz/blog/index.html?demo=GISAlgorithm)

代码地址：[GISAlgorithm Code](http://gisarmory.xyz/blog/index.html?source=GISAlgorithm)


---

原文地址：http://gisarmory.xyz/blog/index.html?blog=GISAlgorithm

关注《[GIS兵器库](http://gisarmory.xyz/blog/index.html?blog=wechat)》， 只给你网上搜不到的GIS知识技能。

![](http://blogimage.gisarmory.xyz/20200923063756.png)

本文章采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议 ](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh)进行许可。欢迎转载、使用、重新发布，但务必保留文章署名《GIS兵器库》（包含链接：  [http://gisarmory.xyz/blog/](http://gisarmory.xyz/blog/)），不得用于商业目的，基于本文修改后的作品务必以相同的许可发布。



# 贡献者

- [@gisarmory](https://github.com/gisarmory)
- [@四季留歌](https://github.com/onsummer)

