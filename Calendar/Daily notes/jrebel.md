---
title: jrebel
date created: 2022-12-07
date modified: 2024-05-29
tags: []
---

## 简介

[使用 Jrebel，实现热部署 | 小决的专栏](https://jueee.github.io/2020/08/2020-08-13-%E4%BD%BF%E7%94%A8Jrebel%EF%BC%8C%E5%AE%9E%E7%8E%B0%E7%83%AD%E9%83%A8%E7%BD%B2/)  
![](http://image.clickear.top/20221207155415.png)

## 安装

安装可参考(简单来说，就是在idea插件中，搜索进行安装):  
[IDEA JRebel插件热部署](https://blog.csdn.net/weixin_43939924/article/details/115591847)

## 激活

最新版激活，会碰到无法连接的问题。可能问题:

1. https服务不支持，不确定是自身代理问题，还是jrebel插件问题
2. 443端口不可用
3. 三方服务，暂无法激活，不确定是不是被拉入黑名单了。

### 三方激活服务

+ [### 捡个便宜 - 交朋友吧 ###](https://www.jpy.wang/page/jrebel.html)(推荐，可用)
+ http://jrebel-license.jiweichengzhu.com/
+ https://jrebel.qekang.com/?utm_source=ld246.com  
优点:

1. 可直接使用。  
缺点:
1. 在最新版idea中存在无法激活的情况。  
![](http://image.clickear.top/20221207160133.png)
> [!QUESTION] jrebel LS client not configured  
> 暂不清楚，为什么不可用。

### 自行搭建激活服务

代码: [jrebel激活服务源码](https://github.com/clickear/JrebelLicenseServerforJava)  
自建服务: [自建jrebel服务](http://jrebel-license.clickear.top/) http://jrebel-license.clickear.top/ 可用（PS：请勿跳转到https，或者使用时，链接修改成http）

> [!TIP] 技巧💡  
> 使用注意:
> 1. 最新版不支持443端口，会报 403 forbiden. http://xxx:443
> 2. 使用https，会报 连接失败 如 https://xxxx
> 3. 如只是本地使用，可直接 http://127.0.0.1:8081/xxxx 即可
> 4. 正确示例: http://jrebel-license.clickear.top:80/5642b2b6-ac60-457f-8186-56c1af40eb4b

#### 使用docker

[在线可以](https://www.jpy.wang/page/jrebel.html)

[jrebel-server docker镜像](https://hub.docker.com/r/cliod/jrebel-server-go)

docker run -d --name jrebel_proxy -p 8888:8888 wangdxing/golang-reverseproxy

```shell
docker run --restart=always -dit --name=jreber1 -p 7001:7001 cliod/jrebel-server-go
```

## 使用中的坑

### 修改类的字段之后，fastjson toString 会报错

[jrebel修改后调用toString方法报错问题 · Issue #2231 · alibaba/fastjson · GitHub](https://github.com/alibaba/fastjson/issues/2231)  
可通过 禁用asm，当前暂无比较快的自动处理方案

```java
SerializeConfig.getGlobalInstance().setAsmEnable(false);
```

### xml修改之后，mybatis不会立即生效

可参考 idea中 jrebel-mybatis-extension插件  
[GitHub - clickear/jrebel-mybatisplus: A hook plugin for Support MybatisPlus that reloads modified SQL maps.](https://github.com/clickear/jrebel-mybatisplus)
