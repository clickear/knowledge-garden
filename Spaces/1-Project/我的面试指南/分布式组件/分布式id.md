---
date created: 2022-09-15
date modified: 2022-09-15
title: 分布式id
---

> [!TIP] 技巧💡
> 1. 主要特点就是要唯一 + 有序（至少整体的递增趋势）
> 2. 全局递增:
> 	3. 数据库db方案，采用双buffer + db号段方案，
> 	4. 典型使用 leaf, TinyId
> 3. 非全局递增，递增趋势的，适合订单id  
> 	雪花算法（时钟回拨，workder没回收）  
> 	时钟回拨:
> 	  1. 采用等待时间
> 	  2. leaf采用，超过太长时间就启动失败
> 	  3. bufferfly， 启动时间戳采用的是“历史时间”，每次请求只增序列值，序列值增满，然后“历史之间”增1，序列值重新计算。  
> 	 workerId回收分配问题
> 	 1. leaf-snowflower，zk顺序持久节点。
> 	 2. bufferfly的zk方案，使用zk持久顺序节点，并记录过期时间。客户端定时续签。
> 	 3. bufferly的db方案，启动时获取失效并且workerId最小的，记录数据库的过期时间，并定时续签。
>

### 1.基于UUID（无序，pass）

介绍：使用jdk内置的api

优点：使用简单，为jdk内置的唯一id生成器

缺点：

- 数据为字符
- 无序化

### 2.数据库自增ID（db瓶颈）

介绍：使用数据库的自增id

优点：无序额外开发，使用简单

缺点：

- 性能不高：瓶颈为DB
- 单点问题：DB宕机，则发号器不可用

### 3.数据库多主模式ID

介绍：采用多主的数据库的自增id，多主之间进行分片切分采用不同步长

优点：解决数据库自增的单点问题，并无需额外开发，使用简单

缺点：

- 扩容复杂
- 性能不高：瓶颈仍然是DB

### 4.号段+DB获取

介绍：每次从DB获取数据，采用每次获取一个范围，获取完再更新DB

优点：对DB压力小

缺点：

- 处理方式需要单独开发
- 性能不是很高：数据批量获取完毕后，再次获取有阻塞问题

### 5.双Buffer+DB号段获取

介绍：基于号段获取中未使用完毕时候异步刷新下次号段

优点：解决号段获取中的性能不是很高问题，可以保持较高的性能

缺点：

- 可靠性不高：数据获取依赖于DB，虽然有号段缓存，但是如果在短时间内没有启动，则还是有问题
- 超高并发度支持度：对于一些超高并发度情况下，号段的使用还是会存在网络的数据消耗。不过这个可以通过动态调整号段大小解决，但是动态配置号段过长，则号段浪费也是问题

### 6.Redis

介绍：通过redis的incr命令实现原子自增操作

优点：使用简单

缺点：

- 重复问题：如果采用RDB方式持久化，则宕机后重启会有数据重复问题，这种可采用AOF的同步每次更新，但这种性能较差
- 重启时长问题：采用RDB有重复这种最基本问题，那么采用AOF，这里宕机后重启时间过长

### 7.Twitter的雪花算法

介绍：通过将64bit中的不同bit划分给一些数据（时间、机器id和自增域），进而保证64bit生成的数据唯一

优点：无网络消耗，性能很高

缺点：

- 时间回拨问题：论文中表述的是采用实际时间，而实际时间有时候会采用历史时间，即使用过去的一些时间，这样就会造成数据重复
- 机器id如何分配问题：机器id需要单独分配，业内目前采用zookeeper和db，但是没有合理的分配和回收方案，存在浪费问题。
- 机器id占有过少：默认的划分为12个，也就是说一个集群中最多为4096台机器做一个业务，其实也不算少了，但是对共享同一个业务的场景就不再适合

```java
package com.baomidou.mybatisplus.core.toolkit;

import java.lang.management.ManagementFactory;
import java.net.InetAddress;
import java.net.NetworkInterface;
import java.util.concurrent.ThreadLocalRandom;

import org.apache.ibatis.logging.Log;
import org.apache.ibatis.logging.LogFactory;

/**
 * 分布式高效有序ID生产黑科技(sequence)
 * <p>优化开源项目：https://gitee.com/yu120/sequence</p>
 *
 * @author hubin
 * @since 2016-08-18
 */
public class Sequence {

    private static final Log logger = LogFactory.getLog(Sequence.class);
    /**
     * 时间起始标记点，作为基准，一般取系统的最近时间（一旦确定不能变动）
     */
    private final long twepoch = 1288834974657L;
    /**
     * 机器标识位数
     */
    private final long workerIdBits = 5L;
    private final long datacenterIdBits = 5L;
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    private final long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    /**
     * 毫秒内自增位
     */
    private final long sequenceBits = 12L;
    private final long workerIdShift = sequenceBits;
    private final long datacenterIdShift = sequenceBits + workerIdBits;
    /**
     * 时间戳左移动位
     */
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    private final long workerId;

    /**
     * 数据标识 ID 部分
     */
    private final long datacenterId;
    /**
     * 并发控制
     */
    private long sequence = 0L;
    /**
     * 上次生产 ID 时间戳
     */
    private long lastTimestamp = -1L;

    public Sequence() {
        this.datacenterId = getDatacenterId(maxDatacenterId);
        this.workerId = getMaxWorkerId(datacenterId, maxWorkerId);
    }

    /**
     * 有参构造器
     *
     * @param workerId     工作机器 ID
     * @param datacenterId 序列号
     */
    public Sequence(long workerId, long datacenterId) {
        Assert.isFalse(workerId > maxWorkerId || workerId < 0,
            String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        Assert.isFalse(datacenterId > maxDatacenterId || datacenterId < 0,
            String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        this.workerId = workerId;
        this.datacenterId = datacenterId;
    }

    /**
     * 获取 maxWorkerId
     */
    protected static long getMaxWorkerId(long datacenterId, long maxWorkerId) {
        StringBuilder mpid = new StringBuilder();
        mpid.append(datacenterId);
        String name = ManagementFactory.getRuntimeMXBean().getName();
        if (StringUtils.isNotBlank(name)) {
            /*
             * GET jvmPid
             */
            mpid.append(name.split(StringPool.AT)[0]);
        }
        /*
         * MAC + PID 的 hashcode 获取16个低位
         */
        return (mpid.toString().hashCode() & 0xffff) % (maxWorkerId + 1);
    }

    /**
     * 数据标识id部分
     */
    protected static long getDatacenterId(long maxDatacenterId) {
        long id = 0L;
        try {
            InetAddress ip = InetAddress.getLocalHost();
            NetworkInterface network = NetworkInterface.getByInetAddress(ip);
            if (network == null) {
                id = 1L;
            } else {
                byte[] mac = network.getHardwareAddress();
                if (null != mac) {
                    id = ((0x000000FF & (long) mac[mac.length - 1]) | (0x0000FF00 & (((long) mac[mac.length - 2]) << 8))) >> 6;
                    id = id % (maxDatacenterId + 1);
                }
            }
        } catch (Exception e) {
            logger.warn(" getDatacenterId: " + e.getMessage());
        }
        return id;
    }

    /**
     * 获取下一个 ID
     *
     * @return 下一个 ID
     */
    public synchronized long nextId() {
        long timestamp = timeGen();
        //闰秒
        if (timestamp < lastTimestamp) {
            long offset = lastTimestamp - timestamp;
            if (offset <= 5) {
                try {
                    wait(offset << 1);
                    timestamp = timeGen();
                    if (timestamp < lastTimestamp) {
                        throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
                    }
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            } else {
                throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
            }
        }

        if (lastTimestamp == timestamp) {
            // 相同毫秒内，序列号自增
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                // 同一毫秒的序列数已经达到最大
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            // 不同毫秒内，序列号置为 1 - 3 随机数
            sequence = ThreadLocalRandom.current().nextLong(1, 3);
        }

        lastTimestamp = timestamp;

        // 时间戳部分 | 数据中心部分 | 机器标识部分 | 序列号部分
        return ((timestamp - twepoch) << timestampLeftShift)
            | (datacenterId << datacenterIdShift)
            | (workerId << workerIdShift)
            | sequence;
    }

    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    protected long timeGen() {
        return SystemClock.now();
    }
}
```

```java
package com.baomidou.mybatisplus.core.toolkit;

import java.sql.Timestamp;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

/**
 * 高并发场景下System.currentTimeMillis()的性能问题的优化
 *
 * <p>System.currentTimeMillis()的调用比new一个普通对象要耗时的多（具体耗时高出多少我还没测试过，有人说是100倍左右）</p>
 * <p>System.currentTimeMillis()之所以慢是因为去跟系统打了一次交道</p>
 * <p>后台定时更新时钟，JVM退出时，线程自动回收</p>
 * <p>10亿：43410,206,210.72815533980582%</p>
 * <p>1亿：4699,29,162.0344827586207%</p>
 * <p>1000万：480,12,40.0%</p>
 * <p>100万：50,10,5.0%</p>
 *
 * @author hubin
 * @since 2016-08-01
 */
public class SystemClock {

    private final long period;
    private final AtomicLong now;

    private SystemClock(long period) {
        this.period = period;
        this.now = new AtomicLong(System.currentTimeMillis());
        scheduleClockUpdating();
    }

    private static SystemClock instance() {
        return InstanceHolder.INSTANCE;
    }

    public static long now() {
        return instance().currentTimeMillis();
    }

    public static String nowDate() {
        return new Timestamp(instance().currentTimeMillis()).toString();
    }

    private void scheduleClockUpdating() {
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(runnable -> {
            Thread thread = new Thread(runnable, "System Clock");
            thread.setDaemon(true);
            return thread;
        });
        scheduler.scheduleAtFixedRate(() -> now.set(System.currentTimeMillis()), period, period, TimeUnit.MILLISECONDS);
    }

    private long currentTimeMillis() {
        return now.get();
    }

    private static class InstanceHolder {
        public static final SystemClock INSTANCE = new SystemClock(1);
    }
}
```

### 基于[[zooKeeper]]的实现

原理: 临时顺序节点

[mama-buy/WorkerIDSenquence.java at 6c9821f696734d660f8f85e5b21bad2f1258b717 · sunweiguo/mama-buy · GitHub](https://github.com/sunweiguo/mama-buy/blob/6c9821f696734d660f8f85e5b21bad2f1258b717/mama-buy-keygen-service/src/main/java/com/njupt/swg/mamabuykeygenservice/KeyGen/WorkerIDSenquence.java)

```java
@Value("${zk.host}")
    private String ZkHost ;

    private static final String ZK_PATH = "/snowflake/workID";

    private static CuratorFramework client;

    @PostConstruct
    void initZKNode() throws Exception {
        client = CuratorFrameworkFactory.newClient(ZkHost,new RetryNTimes(10, 5000));
        client.start();
        log.info("zk client start successfully!");
        Stat stat = client.checkExists().forPath(ZK_PATH);
        if (stat==null){
            client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(ZK_PATH);
        }
    }

    public  long getSequence(String hostname) throws Exception {
        if(StringUtils.isBlank(hostname)){
            hostname = "snowflake_";
        }
        String path = client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(ZK_PATH+"/"+hostname);
       // snowflake_0000000000
        long sequence = Long.valueOf(path.substring(path.length()-4,path.length()));
        return sequence;
    }
```

### 8.美团的Leaf

介绍：采用双Buffer+Db方式获取号段，也支持雪花算法获取

优点：

- 双Buffer+DB：
- 解决号段获取中的性能不是很高问题，可以保持较高的性能
- 雪花算法：
- 无网络消耗，性能很高
- 时间回拨问题解决：时间回拨采用等待方式，算是一种解决方案
- 解决workerId分配问题

缺点：

- 双Buffer+DB：
- 可靠性不高：数据获取依赖于DB，虽然有号段缓存，但是如果在短时间内没有启动，则还是有问题
- 超高并发度支持度：对于一些超高并发度情况下，号段的使用还是会存在网络的数据消耗。不过这个可以通过动态调整号段大小解决，但是动态配置号段过长，则号段浪费也是问题
- 雪花算法
- 时间回拨问题：虽然采用等待方式解决，但是如果时钟回拨两次，则会抛出异常
- workerId分配问题：目前采用zookeeper的顺序节点，只增不减（待确认），集群中达到1024台机器，则会有问题

### 9.百度的UidGenerator

介绍：改造雪花算法，将雪花算法中的字段可自定义，进而根据bit进而适配不同的使用场景

优点：

- 无网络消耗，性能很高
- 机器id分配解决：通过将机器id的分配方式让用户自定义解决机器id分配问题
- bit动态化：可以让用户根据自己的场景动态的调整对应的bit，进而适配不同的业务场景

缺点：

- 时间回拨问题：时间回拨处理简单粗暴
- 机器id上限问题：这是个很严重的问题，就是没分配一次就会少一个，虽然上限为22，也就是分配4194304次数就不能再分配了，就会抛出异常

### 10.滴滴的TinyId

介绍：借鉴美团的Leaf框架中的双Buffer+DB方式并在后端集群方面做了改进

优点：

- 解决号段获取中的性能不是很高问题，可以保持较高的性能

缺点：

- 可靠性不高：数据获取依赖于DB，虽然有号段缓存，但是如果在短时间内没有启动，则还是有问题
- 超高并发度支持度：对于一些超高并发度情况下，号段的使用还是会存在网络的数据消耗。不过这个可以通过动态调整号段大小解决，但是动态配置号段过长，则号段浪费也是问题

## 11. Butterfly

前面存在的问题：

1. 时钟回拨问题，
2. workerId分配问题

[Butterfly 简介 · 语雀](https://www.yuque.com/simonalong/butterfly/tul824)

## 参考

[三、业内方案对比 · 语雀](https://www.yuque.com/simonalong/butterfly/lgacnt)

## 参考

[Leaf——美团点评分布式ID生成系统 - 美团技术团队](https://tech.meituan.com/2017/04/21/mt-leaf.html)

[zookeeper分配 · 语雀](https://www.yuque.com/simonalong/butterfly/qwdltm)
