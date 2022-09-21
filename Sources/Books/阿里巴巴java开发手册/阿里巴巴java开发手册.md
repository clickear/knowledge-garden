---
date created: 2022-09-21
date modified: 2022-09-21
title: 阿里巴巴java开发手册
---

	
	
	

+ try-with-resource，释放资源
+ Random，避免被多线程使用。虽然是线程安全，但是会因为竞争同一个seed，导致性能下降。推荐使用Math.random或者ThreadLocalRandom
	+ apache.common中RandomUtil实现: 使用的是Random
	+ hutool中RandomUtil实现: 使用的是 ThreadLocalRandom
+ Instant，记日志用。与时区无关。不能自定义格式输出
	+ LocalDateTime ，可以自定义格式。DateTiemFormtter.ofPattern. 带时区，默认系统时区。
