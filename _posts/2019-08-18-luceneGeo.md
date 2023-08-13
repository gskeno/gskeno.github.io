---
layout: post
title: 地理位置 geo
tags:
- lucene
categories: lucene
description: 地理位置 GEO
---

饿了么，美团外卖中，是如何搜索指定目标周围2公里的店铺的?

# geo

两个主要话题，

- 如何对经纬度(x,y)进行编码，编码结果用`一个字符串`来表达。
- 两个地理位置A,B之间的距离远近，编码后如何判别。

geo hash编码就能解决以上这两个问题，具体分析可见这篇博客[Geo hash](https://www.cnblogs.com/muson/archive/2013/01/31/2883896.html)


# lucene中应用



# 参考
- [Geo hash](https://www.cnblogs.com/muson/archive/2013/01/31/2883896.html)
- [geohash-js](https://github.com/davetroy/geohash-js)
- [lucene geohash 在外卖场景中，商家不规则多边形配送范围技术应用](https://www.exyb.cn/news/show-4617465.html?action=onClick)