---
title: redis-geo
date created: 2023-11-05
date modified: 2023-11-05
---

> [!TIP] 典型应用 LBS，基于位置的服务💡  
>  经纬度位置，使用[[GeoHash]]进行编码后。可以用来标识位置。

## 命令

- GEOADD 命令：用于把一组经纬度信息和相对应的一个 ID 记录到 GEO 类型集合中；
- GEORADIUS 命令：会根据输入的经纬度位置，查找以这个经纬度为中心的一定范围内的其他元素。当然，我们可以自己定义这个范围
- geopos：查询位置信息
- geodist：距离统计
- georadius：查询某位置内的其他成员信息
- geohash：查询位置的哈希值.即[[GeoHash]]值
- zrem：删除地理位置

```shell
GEOADD cars:locations 116.034579 39.030452 33
-- 添加车辆id为33的经纬度信息
GEORADIUS cars:locations 116.054579 39.030452 5 km ASC COUNT 10
-- 查找以这个经纬度为中心的 5 公里内的车辆信息，并返回给 LBS 应用
```
