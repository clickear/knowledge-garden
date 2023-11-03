---
title: redis-list
date created: 2023-11-02
date modified: 2023-11-02
---

## 命令

![image.png](http://image.clickear.top/20231102110149.png)

## 内部编码

[[ziplist]]

## 使用场景

> [!TIP] 技巧💡
> + lpush+lpop=Stack（栈）
> + lpush+rpop=Queue（队列）
> + lpsh+ltrim=Capped Collection（有限集合）
> + lpush+brpop=Message Queue（消息队列）

### [[reids-消息队列]]

### 文章列表，分页查询
