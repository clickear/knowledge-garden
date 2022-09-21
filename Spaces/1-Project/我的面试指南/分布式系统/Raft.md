# 总结


# 常青笔记
+ 通过领导人的方式，Raft 将一致性问题分解成了三个相对独立的子问题：
	-   **领导选举**：当现存的领导人发生故障的时候, 一个新的领导人需要被选举出来
	-   **日志复制**：领导人必须从客户端接收日志条目（log entries）然后复制到集群中的其他节点，并强制要求其他节点的日志和自己保持一致。
	-   **安全性**：在 Raft 中安全性的关键是在图 3 中展示的状态机安全：如果有任何的服务器节点已经应用了一个确定的日志条目到它的状态机中，那么其他服务器节点不能在同一个日志索引位置应用一个不同的指令。章节 5.4 阐述了 Raft 算法是如何保证这个特性的；这个解决方案涉及到选举机制（5.2 节）上的一个额外限制。
+ Raft 算法属于 [[MultiPaxos]] 算法，它是在兰伯特 Multi-Paxos 思想的基础上，做了一些简化和限制，比如增加了**日志必须是连续的**，只支持领导者、跟随者和候选人三种状态，在理解和算法实现上都相对容易许多。
+ 选主过程，基于[[多数派]] 进行选举。也是[[领导者模型]]，即只有1个提案者。解决了活锁问题。当票数相同时，使用**随机超时心跳**。通过**心跳**来检测存活性，如果超时了，则重新发起选择leader。
+ 通过主节点来发起写操作，并进行日志复制。基于[[状态机复制]] 的日志复制来实现各个节点的数据一致性。
+ 是[[AP]]模型
+ [[脑裂]]
+ Raft 算法通过任期、领导者心跳消息、随机选举超时时间、先来先服务的投票原则、大多数选票原则等，保证了一个任期只有一位领导，也极大地减少了选举失败的情况
+ 在 Raft 中，不是所有节点都能当选领导者，只有**日志最完整**的节点，才能当选领导者。并且日志是**顺序**的。
+ 唯一的leader，领导者选举权，需要最新提的日志的副本。日志需要保证连续，日志提交需要靠推进commit index.
+ 成员变更，导致可能会因为网络分区，出现2个leader。 2种方案，1. 每次只新增或者删减1个，这边就每次都能达到大多数。2. 使用联合共识，即1个消息的提交，需要每个分区的大多数都提交。

# 重点摘要
+ 这是一种通过对日志进行管理来达到一致性的算法，其是一种 [[AP]] 的一致性算法。Raft 通过选举 Leader 并由 Leader 节点负责管理日志复制来实现各个节点间数据的一致性。

## 三种角色
在 Raft 中，节点有三种角色： 
### Leader：
唯一负责处理客户端写请求的节点；也可以处理客户端读请求；同时负责日志复制工作。[[状态机复制]]
### Candidate：
Leader 选举的候选人，其可能会成为 Leader 
### Follower：
可以处理客户端读请求；负责同步来自于 Leader 的日志；当接收到其它 Cadidate 的投票请求后可以进行投票；当发现 Leader 挂了，其会转变为 Candidate 发起 Leader 选举。
term，任期，相当于 Paxos 中的 epoch，表示一个新的 leader 上任了。

| 角色      | 内容                                                                                                                  | 备注 |
| --------- | --------------------------------------------------------------------------------------------------------------------- | ---- |
| Leader    | 1.广播appendEntries或者心跳(entry为空) <br> 2. 休眠一段时间，然后继续发送广播                                                                        |      |
| Follower  | 1. 每过6~10个心跳时间获取一次心跳标志(使用LinkBlockQueue.poll) <br> 2. 如果心跳超时（结果为空），进行下一个状态；否则继续循环 |      |
| Candidate | 1.给自己投票:投一票、term+1 <br> 2. candidate发起投票(广播) <br> 3. 如果超过半数，则设置成leader <br> 4. 如果其它节点的任务term更高，则降至为follower                               |      |

### 节点服务器规则
#### 所有服务器：
-   如果`commitIndex > lastApplied`，则 lastApplied 递增，并将`log[lastApplied]`应用到状态机中（5.3 节）
-   如果接收到的 RPC 请求或响应中，任期号`T > currentTerm`，则令 `currentTerm = T`，并切换为跟随者状态（5.1 节）

#### 跟随者
-   响应来自候选人和领导人的请求
-   如果在超过选举超时时间的情况之前没有收到**当前领导人**（即该领导人的任期需与这个跟随者的当前任期相同）的心跳/附加日志，或者是给某个候选人投了票，就自己变成候选人

#### 候选人
-   在转变成候选人后就立即开始选举过程
    -   自增当前的任期号（currentTerm）
    -   给自己投票
    -   重置选举超时计时器
    -   发送请求投票的 RPC 给其他所有服务器
-   如果接收到大多数服务器的选票，那么就变成领导人
-   如果接收到来自新的领导人的附加日志（AppendEntries）RPC，则转变成跟随者
-   如果选举过程超时，则再次发起一轮选举
#### 领导人
-   一旦成为领导人：发送空的附加日志（AppendEntries）RPC（心跳）给其他所有的服务器；在一定的空余时间之后不停的重复发送，以防止跟随者超时（5.2 节）
-   如果接收到来自客户端的请求：附加条目到本地日志中，在条目被应用到状态机后响应客户端（5.3 节）
-   如果对于一个跟随者，最后日志条目的索引值大于等于 nextIndex（`lastLogIndex ≥ nextIndex`），则发送从 nextIndex 开始的所有日志条目：
    -   如果成功：更新相应跟随者的 nextIndex 和 matchIndex
    -   如果因为日志不一致而失败，则 nextIndex 递减并重试
-   假设存在 N 满足`N > commitIndex`，使得大多数的 `matchIndex[i] ≥ N`以及`log[N].term == currentTerm` 成立，则令 `commitIndex = N`（5.3 和 5.4 节）



## 动画
[RAFT动画演示](http://thesecretlivesofdata.com/raft/)
Q&A环节:
1. Q: 怎么选leader？
A: 若 follower 在心跳超时范围内没有接收到来自于 leader 的心跳，则认为 leader 挂了。发起选举 --> 投票 --> 广播结果。
---
Q: 如果票数相同怎么处理？
A: 进行随机超时重新选择。

---

## 选举leader
![](http://image.clickear.top/20220210103739.png)
> 跟随者只响应来自其他服务器的请求。如果跟随者接收不到消息，那么他就会变成候选人并发起一次选举。获得集群中大多数选票的候选人将成为领导者。在一个任期内，领导人一直都会是领导人直到自己宕机了.
Raft 算法实现了**随机超时时间**的特性。也就是说，每个节点等待领导者节点心跳信息的超时时间间隔是随机的。通过上面的图片你可以看到，集群中没有领导者，而节点 A 的等待超时时间最小（150ms），它会最先因为没有等到领导者的心跳信息，发生超时。
这个时候，节点 A 就增加自己的任期编号，并推举自己为候选人，先给自己投上一张选票，然后向其他节点发送请求投票 RPC 消息，请它们选举自己为领导者。


![](http://image.clickear.top/20220211164211.png)

### 投票过程
1. follower递增自己的term. 
2. follower将自己的状态变为candidate. 
3. 投票给自己. 
4. 向集群其它机器发起投票请求(RequestVote请求). 
5. 当以下情况发生, 结束自己的candidate状态. 
	1. 超过集群一半服务器都同意, 状态变为leader, 并且立即向所有服务器发送心跳消息, 之后按照心跳间隔时间发送心跳消息. 任意一个term中的任意一个服务器只能投一次票, 所有的candidate在此term已经投给了自己, 那么需要另外的follower投票才能赢得选举. 
	2. 发现了其它leader并且这个leader的term不小于自己的term, 状态转为follower. 否则丢弃消息. 
	3. 没有服务器赢得选举, 可能是由于网络超时或者服务器原因没有leader被选举, 这种情况比较简单, 超时之后重试. 有一种情况被称为split votes, 比如一个有三个服务器的集群中所有服务器同时发起选举, 那么就不可能有leader被选举出来, 此时如果超时之后重试很可能所有服务器又同时发起选举, 这样永远不可能有leader被选举出来. raft处理这种情况是采用上文提到过的random election timeout, 随机超时保证了split votes发生的几率很小. 
### follower何时会同意
如果发起的投票请求包含的term大于等于当前term, 并且日志信息不旧于candidate的日志信息, 那么会同意. 关于日志的相关信息, 在log replication再讨论

### term如何更新
所有请求和响应的接收方在接收到更大的term时都必须更新自己的term, 这保证了投票最终能够选出一个leader. 

### 节点间是如何通讯的呢？
在 Raft 算法中，服务器节点间的沟通联络采用的是远程过程调用（RPC），在领导者选举中，需要用到这样两类的 RPC：
1. 请求投票（RequestVote）RPC，是由候选人在选举期间发起，通知各节点进行投票；
2. 日志复制（AppendEntries）RPC，是只能由**领导者**发起，用来复制日志和提供心跳消息。
### 什么是任期呢？


### 选举有哪些规则？
1. 与leader的心跳，超时了，发起重新选举。发起投票
2. 在选举中，获得大多数票，会推举为leader
3. 在这个任期内，一直作为leader。直到自身出现问题。其它从节点超时了，进行重新选举
4. 如果投票者的term_id + index 没我新，则拒绝服务。


Raft 算法和兰伯特的 Multi-Paxos 不同之处，主要有 2 点。
1. 首先，在 Raft 中，不是所有节点都能当选领导者，只有**日志最完整**的节点，才能当选领导者；
2. 其次，在 Raft 中，日志必须是连续的。

Raft 算法通过任期、领导者心跳消息、随机选举超时时间、先来先服务的投票原则、大多数选票原则等，保证了一个任期只有一位领导，也极大地减少了选举失败的情况

### 随机超时时间又是什么？
在议会选举中，常出现未达到指定票数，选举无效，需要重新选举的情况。这时候，需要进行

1. **跟随者**等待领导者心跳信息超时的时间间隔，是随机的；
2. 为达到指定票数时，会重新选举，这时候需要等待随机时间。

## 数据同步（日志如何复制）
Raft 算法一致性的实现，是基于[[状态机复制|日志复制状态机]]的。状态机的最大特征是，不同 Server 中的状态机若当前状态相同，然后接受了相同的输入，则一定会得到相同的输出。是一个[[AP]]模型。
### 过程
![同步过程](http://image.clickear.top/20220127002258.png)

![](http://image.clickear.top/20220128172835.png)


当 leader 接收到 client 的写操作请求后，大体会经历以下流程： 1. leader 将数据封装为日志 
1. leader将日志并行发送给所有 follower，然后等待接收 follower 响应 
2. 当 leader 接收到过半响应后，将日志 commit 到自己的状态机，状态机会输出一个结果， 同时日志状态变为了 committed 同时 leader 还会通知所有 follower 
3. 将日志 apply 到它们本地的状态机，日志状态变为了 applied 
4. 在 apply 通知发出的同时，leader 也会向 client 发出成功处理的响应
### [[AP]]模型
![](http://image.clickear.top/20220127002340.png)
日志由有序编号(log index)的日志条目组成. 每个日志条目包含它被创建时的任期号(**term**), 和用于状态机执行的命令（**command**）. 如果一个日志条目被复制到大多数服务器上, 就被认为可以提交(commit)了。为了保证可用性，各个节点中的日志可 以不完全相同，但 leader 会不断给 follower 发送 log，以使各个节点的 log 最终达到相同。 即 raft 算法不是强一致性的，而是[[最终一致性]]的。

Raft日志同步保证如下两点: （特指的是已提交的日志）
1. 如果不同日志中的两个条目有着相同的索引和任期号, 则它们所存储的命令是相同的. 
2. 如果不同日志中的两个条目有着相同的索引和任期号, 则它们之前的所有条目都是完全一样的. 
3. 如果有部分日志未提交，不一致。会被leader覆盖。

### Followers日志不一致怎么处理？
![](http://image.clickear.top/20220128182117.png)
> 总结起来，就是Leader通过日志复制 RPC<span style="background-color:#ffff00"> 一致性检查</span>，找到跟随者节点上与自己相同日志项的最大索引值，然后复制并更新覆盖该索引值之后的日志项，实现了各节点日志的一致。需要你注意的是，跟随者中的不一致日志项会被领导者的日志覆盖，而且领导者从来不会覆盖或者删除自己的日志

上图阐述了一些Followers可能和新的Leader日志不同的情况. 一个Follower可能会丢失掉Leader上的一些条目, 也有可能包含一些Leader没有的条目, 也有可能两者都会发生. 丢失的或者多出来的条目可能会持续多个任期. 
Leader通过强制Followers复制它的日志来处理日志的不一致, Followers上的不一致的日志会被Leader的日志覆盖。
Leader为了使Followers的日志同自己的一致, Leader需要找到Followers同它的日志一致的地方, 然后覆盖Followers在该位置之后的条目.。
Leader会从后往前试, 每次AppendEntries失败后尝试前一个日志条目, 直到成功找到每个Follower的日志一致位点, 然后向后逐条覆盖Followers在该位置之后的条目


### commitIndex推进
![](http://image.clickear.top/20220210102908.png)
-   CommitIndex `(TermId, LogIndex)`：
    -   所谓 commitIndex，就是已达成多数派，可以应用到状态机的最新的日志位置
    -   日志被复制到 followers 后，先持久化，并不能马上被应用到状态机
    -   只有 leader 知道日志是否达成多数派，是否可以应用到状态机
    -   Followers 记录 leader 发来的当前 commitIndex，所有小于等于 commitIndex 的日志均可以应用到状态机
-   CommitIndex推进：
    -   Leader 在下一个 AppendEntries RPC (也包括 Heartbeat)中携带当前的 commitIndex
    -   Followers 检查日志有效性通过则接受 AppendEntries 并同时更新本地 commitIndex，最后把所有小于等于 commitIndex 的日志应用到状态机

-   commit理解
-   **Leader：Raft不会通过计算备份的数目，来提交****之前term****的log entry；只有leader的****当前term****的log entry才会计算备份数并committed。一旦当前term的entry以这种方式被committed了，那么之前的所有entry都将因为Log Matching Property而被间接committed。
-   **Follower：由leader在下一次append entries时，follower才提交。**

## 安全性
Raft增加了如下两条限制以保证安全性: 
+ ==拥有最新的已提交的log entry的Follower才有资格成为Leader==. 
这个保证是在RequestVote RPC中做的, Candidate在发送RequestVote RPC时, 要带上自己的最后一条日志的term和log index, 其他节点收到消息时, 如果发现自己的日志比请求中携带的更新, 则拒绝投票. 日志比较的原则是, 如果本地的最后一条log entry的term更大, 则term大的更新, 如果term一样大, 则log index更大的更新. 
+ Leader只能推进commit index来提交当前term的已经复制到大多数服务器上的日志, 旧term日志的提交要等到提交当前term的日志来间接提交(log index 小于 commit index的日志被间接提交)
> 如果提交的日志记录是前一任期的, 则必须跟随本届任期的日志记录一起提交才算安全提交 [raft协议几点更正和补充 - 知乎](https://zhuanlan.zhihu.com/p/39105353)

![](http://image.clickear.top/20220128182459.png)

在阶段a, term为2, S1是Leader, 且S1写入日志(term, index)为(2, 2), 并且日志被同步写入了S2; 
在阶段b, S1离线, 触发一次新的选主, 此时S5被选为新的Leader, 此时系统term为3, 且写入了日志(term, index)为(3,  2);
S5尚未将日志推送到Followers就离线了, 进而触发了一次新的选主, 而之前离线的S1经过重新上线后被选中变成Leader, 此时系统term为4, 此时S1会将自己的日志同步到Followers, 按照上图就是将日志(2,  2)同步到了S3, 而此时由于该日志已经被同步到了多数节点(S1, S2, S3), 因此, 此时日志(2, 2)可以被提交了. ; 

在阶段d, S1又下线了, 触发一次选主, 而S5有可能被选为新的Leader(这是因为S5可以满足作为主的一切条件: 1. term = 5 > 4, 2. 最新的日志为(3, 2), 比大多数节点(如S2/S3/S4的日志都新), 然后S5会将自己的日志更新到Followers, 于是S2, S3中已经被提交的日志(2, 2)被截断了. 
增加上述限制后, 即使日志(2, 2)已经被大多数节点(S1, S2, S3)确认了, 但是它不能被提交, 因为它是来自之前term(2)的日志, 直到S1在当前term(4)产生的日志(4,  4)被大多数Followers确认, S1方可提交日志(4, 4)这条日志, 当然, 根据Raft定义, (4, 4)之前的所有日志也会被提交. 此时即使S1再下线, 重新选主时S5不可能成为Leader, 因为它没有包含大多数节点已经拥有的日志(4, 4).
> 个人理解为，如果未提交到大多数节点，也就不算commit。


## 日志压缩
在实际的系统中, 不能让日志无限增长, 否则系统重启时需要花很长的时间进行回放, 从而影响可用性. Raft采用对整个系统进行[[快照 |snapshot]]来解决, snapshot之前的日志都可以丢弃. 

每个副本独立的对自己的系统状态进行snapshot, 并且只能对已经提交的日志记录进行snapshot. 
Snapshot中包含以下内容: 
+ 日志元数据. 最后一条已提交的 log entry的 log index和term. 这两个值在snapshot之后的第一条log entry的AppendEntries RPC的完整性检查的时候会被用上. 
+ 系统当前状态. 
当Leader要发给某个日志落后太多的Follower的log entry被丢弃, Leader会将snapshot发给Follower. 或者当新加进一台机器时, 也会发送snapshot给它. 发送snapshot使用InstalledSnapshot RPC(RPC细节参见八, Raft算法总结). 

做snapshot既不要做的太频繁, 否则消耗磁盘带宽,  也不要做的太不频繁, 否则一旦节点重启需要回放大量日志, 影响可用性. 推荐当日志达到某个固定的大小做一次snapshot. 

做一次snapshot可能耗时过长, 会影响正常日志同步. 可以通过使用copy-on-write技术避免snapshot过程影响正常日志同步. 

## 成员变更 #uncomplete 
成员变更是在集群运行过程中副本发生变化, 如增加/减少副本数, 节点替换等. 

成员变更也是一个分布式一致性问题, 既所有服务器对新成员达成一致. 但是成员变更又有其特殊性, 因为在成员变更的一致性达成的过程中, 参与投票的进程会发生变化. 

如果将成员变更当成一般的一致性问题, 直接向Leader发送成员变更请求, Leader复制成员变更日志, 达成多数派之后提交, 各服务器提交成员变更日志后从旧成员配置(Cold)切换到新成员配置(Cnew). 

因为各个服务器提交成员变更日志的时刻可能不同, 造成各个服务器从旧成员配置(Cold)切换到新成员配置(Cnew)的时刻不同. 

成员变更不能影响服务的可用性, 但是成员变更过程的某一时刻, 可能出现在Cold和Cnew中同时存在两个不相交的多数派, 进而可能选出两个Leader, 形成不同的决议, 破坏安全性

+ 成员变更的问题，主要在于进行成员变更时，可能存在新旧配置的 2 个“大多数”，导致集群中同时出现两个领导者，破坏了 Raft 的领导者的唯一性原则，影响了集群的稳定运行。
+ 单节点变更是利用“一次变更一个节点，不会同时存在旧配置和新配置 2 个‘大多数’”的特性，实现成员变更。
+ 因为**联合共识**实现起来复杂，不好实现，所以绝大多数 Raft 算法的实现，采用的都是单节点变更的方法（比如 Etcd、Hashicorp Raft）。其中，Hashicorp Raft 单节点变更的实现，是由 Raft 算法的作者迭戈·安加罗（Diego Ongaro）设计的，很有参考价值


## Raft与[[MultiPaxos]]的异同
Raft与Multi-Paxos都是基于领导者的一致性算法, 乍一看有很多地方相同, 下面总结一下Raft与Multi-Paxos的异同. 

Raft与Multi-Paxos中相似的概念: 

![](http://image.clickear.top/20220129100843.png)

![](http://image.clickear.top/20220129100914.png)



## 演进

![[分布式算法#paxos的算法演进]]


### Multipaxos、ZAB、Raft为什么是等价算法？
![[分布式算法#MultiPaxos 、 ZAB 、 Raft 为什么是等价算法？]]


## Leader宕机
总原则: 
1. 客户端重试（处理未成功）
2. 有点类似数据库的二阶段提交。先prepare，在commit。
	1. 有数据能准确同步时，尽量保证业务进行。
		1. 数据未同步，重新选择，则进行数据丢弃，客户端重试
		2. 数据同步时，部分同步成功。如果leader是同步成功的节点，则继续同步。否则因为lader无法继续任务，走丢弃流程
		3. apply 通知发出后 Leader 挂了，说明leader已经处理了请求，无需再次处理。

### （1） 请求到达前 Leader 挂了
client 发送写操作请求到达 Leader 之前 Leader 就挂了，因为请求还没有到达集群，所以 这个请求对于集群来说就没有存在过，对集群数据的一致性没有任何影响。Leader 挂了之 后，会选举产生新的 Leader。
由于 Stale Leader（失效的 Leader）并未向 client 发送成功处理响应，所以 client 会重新 发送该写操作请求（若 Client 具有重试机制的话)
### （2） 未开始同步数据前 Leader 挂了
client 发送写操作请求给 Leader，请求到达 Leader 后，Leader 还没有开始向 Followers 复制数据 Leader 就挂了。这时集群会选举产生新的 Leader，Stale Leader 重启后会作为Follower 重新加入集群，并同步新 Leader 中的数据以保证数据一致性。之前接收到 client 的 数据被丢弃。 由于 Stale Leader 并未向 client 发送成功处理响应，所以 client 会重新发送该写操作请求 （若 Client 具有重试机制的话）
### （3） 同步完部分后 Leader 挂了
client 发送写操作请求给 Leader，Leader 接收完数据后开始向 Follower 复制数据。在部分 Follower 复制完后 Leader 挂了（可以过半也可以不过半）。由于 Leader 挂了，就会发起新 的 Leader 选举。 
+ 若 Leader 产生于已经复制完日志的 Follower，其会继续将前面接收到的写操作请求完成，并向 client 进行响应。 
+ 若 Leader 产生于尚未复制日志的 Follower，那么原来已经复制过日志的 Follower 则会将这个没有完成的日志放弃。由于 client 没有接收到响应，所以 client 会重新发送该写操作请求（若 Client 具有重试机制的话）。

### （4） apply 通知发出后 Leader 挂了
client 发送写操作请求给 Leader，Leader 接收完数据后开始向 Follower 复制数据。Leader 成功接收到过半 Follower 复制完毕的响应后，Leader 将日志写入到状态机。此时 Leader 向 Follower 发送 apply 通知。在发送通知的同时，也会向 client 发出响应。此时 leader 挂了。
由于 Stale Leader 已经向 client 发送成功接收响应，且 apply 通知已经发出，说明这个写操作请求已经被 server 成功处理。


# 源码学习
 [GitHub - DanielJyc/raft-simple: raft协议的Java版本简单实现](https://github.com/DanielJyc/raft-simple)

# 资料
+ [raft-zh_cn/raft-zh_cn.md at master · maemual/raft-zh_cn · GitHub](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md) ⭐
+ [Raft协议详解 - 知乎](https://zhuanlan.zhihu.com/p/27207160) ⭐
+ [Raft算法详解 - 知乎](https://zhuanlan.zhihu.com/p/32052223) ⭐
+ [raft算法浅析 - 知乎](https://zhuanlan.zhihu.com/p/38779730) ⭐
+  [raft协议几点更正和补充 - 知乎](https://zhuanlan.zhihu.com/p/39105353) ⭐
+ [SOFAJRaft 介绍 · SOFAStack](https://www.sofastack.tech/projects/sofa-jraft/overview/)
+ [GitHub - DanielJyc/raft-simple: raft协议的Java版本简单实现](https://github.com/DanielJyc/raft-simple)
