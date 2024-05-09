---
title: redis-数据压缩调研
date created: 2024-04-24
date modified: 2024-04-24
---

[[wiki:121167489]]  
结合数据分布，序列化/压缩方式的效率，效果，易用性，扩展性，最适合使用的方式是protostuff和fst序列化，如果有对压缩比要求很高的数据，可以结合zstd或lz4压缩使用.  
[[wiki: 126325859]] 压缩效果
