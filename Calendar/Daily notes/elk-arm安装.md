---
title: elk-arm安装
date created: 2023-11-15
date modified: 2024-06-07
---

## 8.11.1-arm64

```shell
 docker pull docker.elastic.co/kibana/kibana:8.11.1-arm64
 docker pull docker.elastic.co/logstash/logstash:8.11.1-arm64
 docker pull docker.elastic.co/kibana/kibana:8.11.1-arm64
```

```shell
# elastic
docker run -d --name elasticsearch --network elk -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -it -m 1GB  docker.elastic.co/elasticsearch/elasticsearch:8.11.1-arm64

# kibana
docker run -d --name kibana --network elk -p 5601:5601 docker.elastic.co/kibana/kibana:8.11.1-arm64

# logstash
docker run -d --name logstash --network elk docker.elastic.co/logstash/logstash:8.11.1-arm64


```

## 7.17.21

```bash
docker network create elastic
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.17.21
docker run -d --name es01-test --net elastic -p 127.0.0.1:9200:9200 -p 127.0.0.1:9300:9300 -e "discovery.type=single-node"  -v /usr/share/elasticsearch/data
docker.elastic.co/elasticsearch/elasticsearch:7.17.21

```

```bash
docker pull docker.elastic.co/kibana/kibana:7.17.21
docker run -d --name kib01-test --net elastic -p 127.0.0.1:5601:5601 -e "ELASTICSEARCH_HOSTS=http://es01-test:9200" docker.elastic.co/kibana/kibana:7.17.21
```

## 资料

[docker elk arm64-掘金](https://juejin.cn/s/docker%20elk%20arm64)  
[Site Unreachable](https://blog.csdn.net/whatfuswd/article/details/133770213)
