---
date created: 2022-09-15
date modified: 2022-09-15
title: zooKeeper
---

## ZooKeeper是什么？

![](http://image.clickear.top/20211030164918.png)

### 特点: 文件系统 + 监听机制

Zookeeper维护一个类似文件系统的树状数据结构，这种特性使得 Zookeeper 不能用于存放大量的数据，每个节点的存放数据上限为1M

Watcher 监听机制是 Zookeeper 中非常重要的特性，我们基于 Zookeeper 上创建的节点，可以对这些节点绑定监听事件，比如可以监听节点数据变更、节点删除、子节点状态变更等事件，通过这个事件机制，可以基于 Zookeeper 实现分布式锁、集群管理等功能。

Watcher 特性：当数据发生变化的时候， Zookeeper 会产生一个 Watcher 事件，并且会发送到客户端。但是客户端只会收到一次通知。如果后续这个节点再次发生变化，那么之前设置 Watcher 的客户端不会再次收到消息。（Watcher 是一次性的操作）。可以通过循环监听去达到永久监听效果。

### 作用（顶级开源项目怎么使用？）

### 元数据存储

Dubbo：ZooKeeper作为注册中心

Kafka：分布式集群的集中式元数据存储，分布式协调和通知

### Master选举

HDFS：Master选举实现HA架构

Canal、condis：分布式集群的集中式元数据存储，Master选举实现HA架构

### 分布式协调

Dubbo、Spring Cloud把系统拆分成很多的服务或者是子系统

ZooKeeper分布式锁

## curator客户端框架

优秀案例，参考dubbo的使用。(`org.apache.dubbo.remoting.zookeeper` 包)

dubbo基于curator框架，简单的封装了一层简单的应用。有support层的abstract接口，是为了前期兼容zk-client框架。后续版本仅支持curator客户端框架。

![http://image.clickear.top/20211030165405.png](http://image.clickear.top/20211030165405.png)

ZookeeperTransporter，主要用来根据配置来获取ZookeeperClient实例，会有缓存来复用。

ZookeeperClient: 封装的zookeeper操作接口。支持创建节点、删除节点、添加监听器等

- DataListener 数据变更，内部通过`TreeCache`来实现
- StateListener 连接状态变更，即连接状态变更，基于curator的`ConnectionStateListener`实现，与curator相比，新增了*`NEW_SESSION_CREATED` 状态。*
- ChildListener 监听子节点的变更情况。通过curator来`CuratorWatcher` 来实现

### Watcher监听机制（标准的事件通知zkclient自带）

#### watcher机制

在ZooKeeper中，接口类Watcher用于表示一个标准的事件处理器，其定义了事件通知相关的逻辑，包含KeeperState和EventType两个枚举类，分别代表了通知状态和事件类型。

Watcher接口定义了事件的回调方法：process（WatchedEvent event）

```java
// dubbo org.apache.dubbo.remoting.zookeeper.curator 的ChildListener实现
Watcher w = new Watcher() {
    @Override
    public void process(WatchedEvent watchedEvent) {
            // if client connect or disconnect to server, zookeeper will queue
            // watched event(Watcher.Event.EventType.None, .., path = null).
            if (event.getType() == Watcher.Event.EventType.None) {
                return;
            }
						// 因为watcher消费一次就没了，需要重新注册
						client.getChildren().usingWatcher(this).forPath(path);
    }
};
```

#### 缓存方式监听(Curator独有)

##### NodeCache 节点缓存的监听

```java
NodeCache nodeCache =
        new NodeCache(client, workerPath, false);
NodeCacheListener l = new NodeCacheListener() {
    @Override
    public void nodeChanged() throws Exception {
        ChildData childData = nodeCache.getCurrentData();
        log.info("ZNode节点状态改变, path={}", childData.getPath());
        log.info("ZNode节点状态改变, data={}", new String(childData.getData(), "Utf-8"));
        log.info("ZNode节点状态改变, stat={}", childData.getStat());
    }
};
nodeCache.getListenable().addListener(l);
nodeCache.start();

// 第1次变更节点数据
client.setData().forPath(workerPath, "第1次更改内容".getBytes());
Thread.sleep(1000);

// 第2次变更节点数据
client.setData().forPath(workerPath, "第2次更改内容".getBytes());
```

##### PathChildrenCache 子节点监听

> Path Cache 用来监听ZNode的子节点事件，包括added、updateed、removed，Path Cache会同步子节点的状态，产生的事件会传递给注册的PathChildrenCacheListener

无法对监听路径所在节点进行监听(即不能监听path对应节点的变化) 只能监听path对应节点下一级目录的子节点的变化内容(即只能监听/path/node1的变化，而不能监听/path/node1/node2 的变化) PathChildrenCache在调用start()方法时，有3种启动模式，分别为 NORMAL-初始化缓存数据为空 BUILD_INITIAL_CACHE-在start方法返回前，初始化获取每个子节点数据并缓存 POST_INITIALIZED_EVENT-在后台异步初始化数据完成后，会发送一个INITIALIZED初始化完成事件

```java
public class Watcher {
    public static void main(String[] args) throws Exception {
        CuratorFramework zkClient = getZkClient();
        String path = "/pathChildrenCache";
        byte[] initData = "initData".getBytes();
        //创建节点用于测试
        zkClient.create().forPath(path, initData);

        PathChildrenCache pathChildrenCache = new PathChildrenCache(zkClient, path, true);
        //调用start方法开始监听 ，设置启动模式为同步加载节点数据
        pathChildrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
        //添加监听器
        pathChildrenCache.getListenable().addListener(new PathChildrenCacheListener() {

            @Override
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
                System.out.println("节点数据变化,类型:" + event.getType() + ",路径:" + event.getData().getPath());
            }
        });
        String childNodePath = path + "/child";
        //创建子节点
        zkClient.create().forPath(childNodePath, "111".getBytes());
        Thread.sleep(1000);
        //更新子节点
        zkClient.setData().forPath(childNodePath, "222".getBytes());
        Thread.sleep(1000);
        //删除子节点
        zkClient.delete().forPath(childNodePath);

        Thread.sleep(Integer.MAX_VALUE);
    }

    private static CuratorFramework getZkClient() {
        String zkServerAddress = "127.0.0.1:2182,127.0.0.1:2183,127.0.0.1:2184";
        ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000, 3, 5000);
        CuratorFramework zkClient = CuratorFrameworkFactory.builder()
                .connectString(zkServerAddress)
                .sessionTimeoutMs(5000)
                .connectionTimeoutMs(5000)
                .retryPolicy(retryPolicy)
                .build();
        zkClient.start();
        return zkClient;
    }

}
```

##### Tree Cache 节点树缓存（包含自身节点和子节点，是NodeCache和PathChildrenCache的结合）

> Path Cache和Node Cache的“合体”，监视路径下的创建、更新、删除事件，并缓存路径下所有孩子结点的数据。

```java
public class Watcher {
    public static void main(String[] args) throws Exception {
        CuratorFramework zkClient = getZkClient();
        String path = "/treeCache";
        byte[] initData = "initData".getBytes();
        //创建节点用于测试
        zkClient.create().forPath(path, initData);

        TreeCache treeCache = new TreeCache(zkClient, path);
        //调用start方法开始监听
        treeCache.start();
        //添加TreeCacheListener监听器
        treeCache.getListenable().addListener(new TreeCacheListener() {

            @Override
            public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
                System.out.println("监听到节点数据变化，类型："+event.getType()+",路径："+event.getData().getPath());
            }
        });
        Thread.sleep(1000);
        //更新父节点数据
        zkClient.setData().forPath(path, "222".getBytes());
        Thread.sleep(1000);
        String childNodePath = path + "/child";
        //创建子节点
        zkClient.create().forPath(childNodePath, "111".getBytes());
        Thread.sleep(1000);
        //更新子节点
        zkClient.setData().forPath(childNodePath, "222".getBytes());
        Thread.sleep(1000);
        //删除子节点
        zkClient.delete().forPath(childNodePath);

        Thread.sleep(Integer.MAX_VALUE);
    }

    private static CuratorFramework getZkClient() {
        String zkServerAddress = "127.0.0.1:2182,127.0.0.1:2183,127.0.0.1:2184";
        ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000, 3, 5000);
        CuratorFramework zkClient = CuratorFrameworkFactory.builder()
                .connectString(zkServerAddress)
                .sessionTimeoutMs(5000)
                .connectionTimeoutMs(5000)
                .retryPolicy(retryPolicy)
                .build();
        zkClient.start();
        return zkClient;
    }

}
```

## 典型运用场景

### Dubbo的服务注册发现

![](http://image.clickear.top/20211030154555.png)

问题:

1. Provider 会把一长串 URL（dubbo://xxx 的字符串）写入到 Zookeeper 里面某个节点里面去。
2. Consumer 的注册也是类似，会写到 Zookeeper 里面某个节点（Consumer 写入的原因，是因为 OPS 服务治理的时候需要实时的消费者数据）。
3. Consumer 发起一个订阅，订阅相关的服务。
4. 当某个服务的 Provider 列表有变化的时候，Zookeeper 会将对应的变化通知到订阅过这个服务的 Consumer 列表

### [[分布式锁]]

### 选举

通过超过半数即是master。
