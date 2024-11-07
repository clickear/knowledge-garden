---
title: ∑ es
date created: 2024-06-06
date modified: 2024-06-14
---

## es 资料

极客时间 Elasticsearch 核心技术与实战

## why  

与工作相关，技术提升。  
想学到什么？  
基本使用，场景，核心原理，集群，调优，最佳实践

## where

1. 官方文档 [What is Elasticsearch? | Elasticsearch Guide [7.17] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/elasticsearch-intro.html)
2. 极客时间 https://b.geekbang.org/member/course/detail/102655
3. wiki 分享
4. 博客 [零基础入门到就业--JAVA篇\_蟑螂恶霸不是恶霸的博客-CSDN博客](https://blog.csdn.net/whatfuswd/category_12170164_6.html)

## what:

领域分层图（自上而下）

+ 设计: 		数据建模实践、索引设计 、 使用场景，routing, index template
+ 应用层 es ： ddl,dml， curd
+ 集成: elk,
+ 应用场景 	 	log  
+ 核心原理	 	 数据索引(倒排索引)，分词， 集群、分布式架构原理、数据存储类型、
+ 存储系统  	[[文档数据库]]、行存储 列存储  
+ 鉴权/安全  
+ 分布式	 	集群/节点 主/副分片
+ 运维/调优配置 分片配置
+ 基础知识 分布式  
	关键源码

细节分层图（由表而里）

+ 数据存储
	+ 定义结构 scheme, mapping
		+ 数据类型, 命名
		+ 动态
	+ 分词器
	+ 数据流走向:
		+ 分布式
			+ 写入
			+ 查询
				+ get --> 简单获取一个文档数据
				+ query --> 复杂查询
			+ redis, 查询、存储
		+ 单机
			+ 逻辑数据模型
			+ 物理数据模型
	+ 底层实现
		+ 索引: b+树 倒排索引
		+ 事务日志(恢复): redolog, translog
		+ 慢日志:
	+ 刷盘策略:

---

+ 基本操作	DML(CRUD) DDL  
+ 接口设计		索引生命周期  
+ 设计原理  
+ 设计方案  
+ 实现源码

## 时间规划: WHEN & HOW 什么时候学，具体怎么学？

6周

官方文档

极客时间  
	概述 4  
	安装 4  
	入门 基本概念 crud 15  
	深入搜索 13  
	分布式 8  
	聚合分析 7  
	数据建模实践、索引设计 7  
	鉴权		3  
	水平扩展 6  
	运维		10  
	实战等	15

## 学习时  

推论是否合理  
一段话总结该知识点

## 学习之后  

是否有基于个人理解的总结？  
是否有项目实战去检验知识？  
最后回顾自己是否拿到了最初期望的结果？

## 复盘效果

## 大纲

版本: 7.17.20

+ [[准实时]] 为什么是准实时? 1秒内？
+ [[文档数据库]]
+ 数据流怎么走？如何节点都可以访问到数据？
+ 存储类型
	+ text: 倒排索引
	+ 数值、geo BKD tree

---

+ 使用
	+ index
		+ 定义
	+ document
		+ crud

### 安装

[[elk-arm安装]]

[6.1 Elasticsearch（一）Docker搭建ES集群\_docker es-CSDN博客](https://blog.csdn.net/whatfuswd/article/details/133770213)

## 资料

+ [[wiki: 270fbea1462f4479b565f8636bc39f8f_80048f64630041e083fefedaad8ec251 技术分享ppt]]
+ [[wiki:156029603 订单索引]]


