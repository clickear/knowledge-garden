---
title: Trees
date created: 2023-06-13
date modified: 2023-06-19
---

参考: [B+树,B-link树,LSM树...一个视频带你了解常用存储引擎数据结构（合集）\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1se4y1U7Dn/?buvid=XU395E039B5DE558C9710D2DB3A55AC3F6F33&is_story_h5=false&mid=H0ejNYFOat6NVNeix6q0Vw%3D%3D&p=1&plat_id=114&share_from=ugc&share_medium=android&share_plat=android&share_session_id=73a47f18-5673-4ea9-8970-0fbbd9b6b0de&share_source=WEIXIN&share_tag=s_i&timestamp=1686585416&unique_k=qp21WwT&up_id=61981458)

![image.png](http://image.clickear.top/20230613001537.png)  
![image.png](http://image.clickear.top/20230613001654.png)

## 存储结构的共性

1. 适合磁盘存储
2. 允许并发操作？

[[B树]]

1. 低高度、高扇出。符合了磁盘存储。
2. B树修改的单位是页，存在[[SMO]]操作，导致B树并发能力不强。

![image.png](http://image.clickear.top/20230613002138.png)

SMO: 分类与合并。  
![image.png](http://image.clickear.top/20230613002404.png)

## [[B+树]]，相对于[[B树]]，加强了1，IO尽量少并且一次读取连续区域。

与[[B树]]类似，但是只有叶子节点存数据，并且叶子节点有双向链表。方便遍历。  
与B树类型，存在SMO，导致了并发操作存在问题。勉强可用。

##[[B-link树 ]]， 加强了2，增删改对存储结构影响尽量小

##[[Bw树 ]]，加强了2，增删改对存储结构影响尽量小.

## [[LSM树]]
