---
date created: 2022-01-26
date modified: 2022-09-16
title: Paxos
---

## 常青笔记

+ 一个是 Basic Paxos 算法，描述的是多节点之间如何就**某个值（提案 Value）**达成共识；
+ 另一个是 Multi-Paxos 思想，描述的是执行多个 Basic Paxos 实例，就**一系列值**达成共识。
+ 在可靠消息通道下，即在不考虑[[拜占庭问题|消息有传递错误]]下，基于[[消息传递]]通讯模型下，分布式系统下，就**某一个**决议达成一致（多个的话，参考[[MultiPaxos]]）
+ prepare阶段(只是咨询，这时候没带提案内容)，是因为有多提案者导致需要预先抢占。多提案者可以加入随机超时，来避免[[活锁]]问题
+ 在accpet阶段，提案内容的选择，是由prepare阶段返回的promise响应来决定的。从多个决策节点的promise结果中，找到最大提案id对应的提案内容，作为提案内容。
	+ 能找到最大提案id，使用决策节点中返回的对应的提案id的提案内容。
	+ 不能找到最大提案id，则使用自己的提案内容。

## 重要摘要

### paxos

Paxos 算法是莱斯利·兰伯特(Leslie Lamport)1990 年提出的一种基于消息传递的、具有高 容错性的一致性算法。Google Chubby 的作者 Mike Burrows 说过，世上只有一种一致性算法， 那就是 Paxos，所有其他一致性算法都是 Paxos 算法的不完整版。Paxos 算法是一种公认的晦 涩难懂的算法，并且工程实现上也具有很大难度。较有名的 Paxos 工程实现有 Google Chubby、 [[ZAB]]、微信的 PhxPaxos 等。

Paxos 算法是用于解决什么问题的呢？Paxos 算法要解决的问题是，**在分布式系统中如何就某个决议达成一致**.也就是，paxos是一种共识算法。  
paxos协议，是基于操作转移进行复制的。是在不考虑  
[[拜占庭问题|消息有传递错误]]的情况下，基于[[消息传递]]通讯模型的

### 算法描述

``` ad-col
这里介绍的算法描述，都是BasicPaxos。其它类MultiPaxos参考paxos算法演进
```

#### 前置条件

+ Paxos 只负责达成一个决议。
+ 唯一的递增的提案id号
+ 一个提案 = 提案id + 提案value。prepare只会带有提案id，申请锁定。
+ 每个表决者在 accept 某提案后，会将该提案的编号 N 记录在本地，这样每个表决者中 保存的已经被 accept 的提案中会存在一个编号最大的提案，其编号假设为 maxN。每个表决者仅会 accept 编号大于自己本地 maxN 的提案。
+ 在众多提案中最终只能有一个提案被选定。
+ 一旦一个提案被选定，则其它服务器会主动同步(Learn)该提案到本地。 没有提案被提出则不会有提案被选定。

#### 角色

在 Paxos 算法中有三种角色，分别具有三种不同的行为。但很多时候，一个进程可能同 时充当着多种角色。

+ Proposer：提案（Proposal）的提案者。在一个集群中，提案者可能存在多个，不同的提案者会提出不同的提案。
+ Acceptor：提案的表决者。即用于表决是否同步某提案。只有过半的 Acceptor 接受了某提案，该提案才会被认为是“选定了”。
+ Learner(也可以理解为记录)：提案的同步者。当提案被选定时，其要在本地执行该提案内容。

#### 流程 ⭐

Paxos 算法包括两个阶段，其中prepare阶段和accept阶段。所以至少需要2次rpc调用。

``` ad-hibox
为什么需要prepare阶段？
是因为basic-paxos是支持多个提案者的，也就是每个提案者只有会有一个值的不同提案。
```

![Paxos 算法整体时序图](http://image.clickear.top/20220127111029.png)

#### prepare阶段

``` ad-hibox
prepare，是为了抢占。每个决策节点本地会记录一个最大的提案号localMaxN。提案节点，发起prepare节点，只接受最新请求。prepare阶段，请求中不待提案内容，只有提案id。只处理最新的提案。旧的直接废弃。接受到prepare后，更新本地的 localMaxN = N;
```

第一阶段“准备”（Prepare）就[相当于上面抢占锁的过程](http://icyfenix.cn/distribution/consensus/paxos.html)。如果某个提案节点准备发起提案，必须先向所有的决策节点广播一个许可申请（称为 Prepare 请求）。提案节点的 Prepare 请求中会附带一个全局唯一的递增的数字 n 作为提案 ID，决策节点收到后，将会给予提案节点两个承诺与一个应答。  
两个承诺是指：

- 承诺不会再接受提案 ID 小于或等于 n 的 Prepare 请求。
- 承诺不会再接受提案 ID 小于 n 的 Accept 请求。  
一个应答是指：
- 不违背以前作出的承诺的前提下，回复已经批准过的提案中 ID 最大的那个提案所设定的值和提案 ID，如果该值从来没有被任何提案设定过，则返回空值。如果违反此前做出的承诺，即收到的提案 ID 并不是决策节点收到过的最大的，那允许直接对此 Prepare 请求不予理会

#### accept阶段

``` ad-hibox
1. accpet阶段，最重要的就是从决策节点的响应中(会返回promise(id,value))，**获取到提案id最大的提案id。然后把对应的maxAcceptValue作为提案内容**。而不管之前的提案内容，这样就可以保证一致性，如果没找到最大的提案id，就以自己为准。
2. 决策节点，接受到accept后，以本地的提案id进行比较，判断是不是之前抢占的。如果是则更新成功。否则就不处理。
3. 当提案节点收到了多数派决策节点的应答（称为 Accepted 应答）后，协商结束，共识决议形成，将形成的决议发送给所有记录节点进行学习

```

当提案节点收到了多数派决策节点的应答（称为 Promise 应答）后，可以开始第二阶段“批准”（Accept）过程，这时有如下两种可能的结果：

- 如果提案节点发现所有响应的决策节点此前都没有批准过该值（即为空），那说明它是第一个设置值的节点，可以随意地决定要设定的值，将自己选定的值与提案 ID，构成一个二元组“(id, value)”，再次广播给全部的决策节点（称为 Accept 请求）。
- 如果提案节点发现响应的决策节点中，已经有至少一个节点的应答中包含有值了，那它就不能够随意取值了，必须无条件地从应答中找出提案 ID 最大的那个值并接受，构成一个二元组“(id, maxAcceptValue)”，再次广播给全部的决策节点（称为 Accept 请求）。  
当每一个决策节点收到 Accept 请求时，都会在不违背以前作出的承诺的前提下，接收并持久化对当前提案 ID 和提案附带的值。如果违反此前做出的承诺，即收到的提案 ID 并不是决策节点收到过的最大的，那允许直接对此 Accept 请求不予理会。  
当提案节点收到了多数派决策节点的应答（称为 Accepted 应答）后，协商结束，共识决议形成

##### 例子

##### 以决策节点返回的最大提案id的内容为准

![X 被选定只取决于 Promise 应答中是否已批准](http://image.clickear.top/20220127104814.png)  
X 被选定为最终值并不是必定需要多数派的共同批准，只取决于 S5提案时 Promise 应答中是否已包含了批准过 X 的决策节点，譬如图 6-3 所示，S5发起提案的 Prepare 请求时，X 并未获得多数派批准，但由于 S3已经批准的关系，最终共识的结果仍然是 X。

---

##### 决策节点的accept，accept的提案id大于本地的localMaxN

![整个系统最终会对“取值为 Y”达成一致](http://image.clickear.top/20220127104538.png)  
S5提案时 Promise 应答中并未包含批准过 X 的决策节点，譬如应答 S5提案时，节点 S1已经批准了 X，节点 S2、S3未批准但返回了 Promise 应答，此时 S5以更大的提案 ID 获得了 S3、S4、S5的 Promise，这三个节点均未批准过任何值，那么 S3将不会再接收来自 S1的 Accept 请求，因为它的提案 ID 已经不是最大的了

### 联动

### basicPaxos支持多提案者，带来了什么问题？

1. 多提案者，带来了并发问题，即可能同时存在多个不同提案。存在并发问题。存在并发，就需要抢占锁。
2. 多个提案者，就存在[[活锁]]问题。在算法实现中会引入**随机超时**时间来避免活锁的产生

### 缺点

### [Paxos 只负责达成一个决议？](https://z.itpub.net/article/detail/48B5D9F7E0D8B72AEB2F0B26FC5818F6)

paxos只负责达成一个提案，一旦达成之后，其他的proposer就没有办法修改了，也就是说其他 proposer 再怎么重复的提交 propose，都只会学习到已经达成一致的 Value 值，然后重复提交：比如说多一个人决议去哪里吃饭，有些人是 proposer，所有人都是 acceptor，这些 proposer 都提除不同的吃饭地点，然后发表提案，最终paxos 达成吃饭地点的提案就可以了。如果是多个不同的提案都需要做，实际上 paxos 并没有定义如何去实现，这是 [[MultiPaxos]] 做的事情

### 活锁问题

活锁指的是任务或者执行者没有被阻塞，由于某些条件没有满足，导致一直重复“尝试 —失败—尝试—失败”的过程。处于活锁的实体是在不断的改变状态，活锁有可能自行解开。  
活锁与死锁的状态有着本质的区别：

1. 活锁是一直在动，是活动状态。
2. 死锁是系统阻塞，是不活动状态

![批准阶段失败，形成活锁](http://image.clickear.top/20220127104344.png)  
P是prepare。 A是accept  
如果两个提案节点交替使用更大的提案 ID 使得准备阶段成功，但是批准阶段失败的话，这个过程理论上可以无限持续下去，形成活锁（Live Lock

### paxos的算法演进 ⭐

![](http://image.clickear.top/20220126145714.png)  
![paxos](http://image.clickear.top/分布式算法(Paxos).png)

## [[MultiPaxos]] 、[[ZAB]]、[[Raft]] 为什么是等价算法？

核心的原理一样

1. 通过随机超时来实现无活锁的选主过程。
2. 通过主节点来发起写操作。同时只有1个主节点。
3. 通过心跳来检测存活性
4. 通过[Quorum](https://www.notion.so/Quorum-f31058d3ebfc4b5bb175b5181196f0a3)机制来保证一致性

[RAFT](https://blog.csdn.net/weixin_34401479/article/details/90588562?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0.pc_relevant_default&spm=1001.2101.3001.4242.1&utm_relevant_index=3)

具体细节上可以有差异：  
譬如是全部节点都能参与选主，还是部分节点能参与，  
譬如Quorum中是众生平等还是各自带有权重，  
譬如该怎样表示值的方式

## 资料

[共识算法 之 Basic Paxos](https://z.itpub.net/article/detail/48B5D9F7E0D8B72AEB2F0B26FC5818F6) ⭐

[可靠分布式系统-paxos的直观解释 - OpenACID Blog](https://blog.openacid.com/algo/paxos/)

