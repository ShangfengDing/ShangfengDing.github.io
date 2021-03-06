---
layout: post
title: Paxos、Raft分布式一致性协议以及Zookeeper
categories: Knowledge
description: 理解 CAP 理论以及 Paxos、Raft 等分布式一致性协议，学习 zookeeper 
keywords: 分布式一致性协议以及Zookeeper
---

学了几次Paxos算法，当时会过后就忘，这次总结整理一下，先从基本的CAP原理复习，之后归纳总结一下两款分布式一致性协议Paxos和Raft，最后记录一下Zookeeper的作用。此篇文章是参考多篇博客整理而出，在此感谢原博主们的精心整理与分享。

## CAP理论

CAP原则又称CAP定理，指的是在一个分布式系统中， Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性），三者不可得兼。CAP原则的精髓就是要么AP，要么CP，要么AC，但是不存在CAP。

- **一致性（C）**：在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）
- **可用性（A）**：在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）
- **分区容忍性（P）**：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。

为什么会这样？我们来看一个简单的问题, 一个DB服务 搭建在两个机房（北京,广州)，两个DB实例同时提供写入和读取。

![不同分区可能在不同worker上](/images/posts/knowledge/zookeeper/cap.PNG)

1. **假设DB的更新操作是同时写北京和广州的DB都成功才返回成功**
在没有出现网络故障的时候，满足CA原则，C 即我的任何一个写入，更新操作成功并返回客户端完成后,分布式的所有节点在同一时间的数据完全一致， A 即我的读写操作都能够成功，但是当出现网络故障时，我不能同时保证CA，即P条件无法满足

2. **假设DB的更新操作是只写本地机房成功就返回，通过binlog/oplog回放方式同步至侧边机房**
这种操作保证了在出现网络故障时,双边机房都是可以提供服务的，且读写操作都能成功，意味着他满足了AP ，但是它不满足C，因为更新操作返回成功后，双边机房的DB看到的数据会存在短暂不一致，且在网络故障时，不一致的时间差会很大（仅能保证最终一致性）

3. **假设DB的更新操作是同时写北京和广州的DB都成功才返回成功且网络故障时提供降级服务**
降级服务，如停止写入，只提供读取功能，这样能保证数据是一致的，且网络故障时能提供服务，满足CP原则，但是他无法满足可用性原则

## Paxos算法

Paxos算法是用来解决分布式系统中，如何就某个值达成一致的算法。它晦涩难懂的程度完全可以跟它的重要程度相匹敌。由于其过于难懂，所以引用他人之前所写博客，在此只对此博客内容进行结论性的总结，如忘记细节还应回原博客学习。

先提一下Paxos算法中的三个角色

* Proposer：议案发起者。
* Acceptor：决策者，可以批准议案。
* Learner：最终决策的学习者。

Paxos要实现的目标的是：  
T1.一次选举必须要选定一个议案（不能出现所有议案都被拒绝的情况）  
T2.一次选举必须只选定一个议案（不能出现两个议案有不同的值，却都被选定的情况）

算法推导：  
P0. 当集群中，超过半数的Acceptor接受了一个议案，那我们就可以说这个议案被选定了（Chosen）。  
P1：一个Acceptor必须接受它收到的第一个议案。  
P2：如果一个值为v的议案被选定了，那么被选定的更大编号的议案，它的值必须也是v。  
P2a：如果一个值为v的议案被选定了，那么Acceptor接受的更大编号的议案，它的值必须也是v。（会出现不满足p1的情况，因此p2b）  
P2b：如果一个值为v的议案被选定了，那么Proposer提出的更大编号的议案，它的值必须也是v。（会出现在路上的最大提案，此时Proposer提出了一个非最大提案的值，重新面临p2a情况，因此应该用p2c方式提出）      
P2c：在所有Acceptor中，任意选取半数以上的Acceptor集合，我们称这个集合为S。Proposal新提出的议案（简称Pnew）必须符合下面两个条件之一：  
  1. 如果S中所有Acceptor都没有接受过议案的话，那么Pnew的编号保证唯一性和递增即可，Pnew的值可以是任意值。  
  2. 如果S中有一个或多个Acceptor曾经接受过议案的话，要先找出其中编号最大的那个议案，假设它的编号为N，值为V。那么Pnew的编号必须大于N，Pnew的值必须等于V。  
在P2c的基础上，还需要有预测未来的能力，当一个议案在提出时（即使已经在发送的半路上了），它必须能够知道当前已经提出的议案的最大编号。做法是：  
议案（n，v）在提出前，必须将自己的编号通知给半数以上的Acceptor。收到通知的Acceptor将n跟自己之前收到的通知进行比较，如果n更大，就将n确认为最大编号。当半数以上Acceptor确认n是最大编号时，议案（n，v）才能正式提交。  
因此Acceptor收到一个新的编号n，在确认n比自己之前收到的编号大时，必须做出承诺（Promise）：不再接受比n小的议案。这个承诺会导致部分漏网之鱼（在发送途中被抢走最大编号的议案），无法形成多数派。

如果我们将半数以上的Acceptor对同一个议案（n，v）做出承诺的状态称作是“锁定”状态。那么这个“锁定”状态具有以下性质：
* 排它性： 所有比n小的议案都不允许提交，已经在途的议案，则不允许其形成多数派。
* 唯一性： 任意时刻，全局只有一个议案能获得“锁定”状态。
* 原子性： 议案n从锁定状态变为非锁定状态的过程是原子的，议案n+1从非锁定状态变更为锁定状态的过程也是原子的。

最后，感谢原文作者利用图片深入浅出的讲解了Paxos算法，让读者可以清晰的明白算法的过程和原理。原文：[分布式理论：深入浅出Paxos算法](https://my.oschina.net/u/150175/blog/2992187)

## 共识算法：Raft

拜占庭将军问题是分布式领域最复杂、最严格的容错模型。但在日常工作中使用的分布式系统面对的问题不会那么复杂，更多的是计算机故障挂掉了，或者网络通信问题而没法传递信息，这种情况不考虑计算机之间互相发送恶意信息，极大简化了系统对容错的要求，最主要的是达到一致性。

所以将拜占庭将军问题根据常见的工作上的问题进行简化：假设将军中没有叛军，信使的信息可靠但有可能被暗杀的情况下，将军们如何达成一致性决定？

对于这个简化后的问题，有许多解决方案，第一个被证明的共识算法是 Paxos，由拜占庭将军问题的作者 Leslie Lamport 在1990年提出，最初以论文难懂而出名，后来这哥们在2001重新发了一篇简单版的论文 Paxos Made Simple，然而还是挺难懂的。

因为 Paxos 难懂，难实现，所以斯坦福大学的教授在2014年发表了新的分布式协议 Raft。与 Paxos 相比，Raft 有着基本相同运行效率，但是更容易理解，也更容易被用在系统开发上。

用拜占庭将军的例子来帮助理解 Raft。假设将军中没有叛军，信使的信息可靠但有可能被暗杀的情况下，将军们如何达成一致性决定？

Raft 的解决方案大概可以理解成 先在所有将军中选出一个大将军，所有的决定由大将军来做。**选举环节**：比如说现在一共有3个将军 A, B, C，每个将军都有一个**随机时间**的倒计时器，倒计时一结束，这个将军就会把自己当成大将军候选人，然后派信使去问其他几个将军，能不能选我为总将军？假设现在将军A倒计时结束了，他派信使传递选举投票的信息给将军B和C，如果将军B和C还没把自己当成候选人（倒计时还没有结束），并且没有把选举票投给其他，他们把票投给将军A，信使在回到将军A时，将军A知道自己收到了足够的票数，成为了大将军。在这之后，是否要进攻就由大将军决定，然后派信使去通知另外两个将军，如果在一段时间后还没有收到回复（可能信使被暗杀），那就再重派一个信使，直到收到回复。

故事先讲到这里，希望不做技术方面的朋友可以大概能理解 Raft 的原理，下面从比较技术的角度讲讲 Raft 的原理。

从拜占庭将军的故事映射到分布式系统上，每个将军相当于一个分布式网络节点，每个节点有三种状态：Follower，Candidate，Leader，状态之间是互相转换的.

每个节点上都有一个倒计时器 (Election Timeout)，时间随机在 150ms 到 300ms 之间。有几种情况会重设 Timeout：
1. 收到选举的请求
2. 收到 Leader 的 Heartbeat

在 Raft 运行过程中，最主要进行两个活动：
1. 选主 Leader Election
2. 复制日志 Log Replication

**选主**：假设现在有5个节点，5个节点一开始的状态都是 Follower。在一个节点倒计时结束 (Timeout) 后，这个节点的状态变成 Candidate 开始选举，它给其他几个节点发送选举请求 (RequestVote)。其他四个节点都返回成功，这个节点的状态由 Candidate 变成了 Leader，并在每个一小段时间后，就给所有的 Follower 发送一个 Heartbeat 以保持所有节点的状态，Follower 收到 Leader 的 Heartbeat 后重设 Timeout。这是最简单的选主情况，**只要有超过一半的节点投支持票了**，Candidate 才会被选举为 Leader，5个节点的情况下，3个节点 (包括 Candidate 本身) 投了支持就行。
                                                                                                                
**复制日志**：Raft 在实际应用场景中的一致性更多的是体现在不同节点之间的数据一致性，客户端发送请求到任何一个节点都能收到一致的返回，当一个节点出故障后，其他节点仍然能以已有的数据正常进行。在选主之后的复制日志就是为了达到这个目的。    
一开始，Leader 和 两个 Follower 都没有任何数据。客户端发送请求给 Leader，储存数据 “sally”，Leader 先将数据写在本地日志，这时候数据还是 Uncommitted (还没最终确认，红色表示)。Leader 给两个 Follower 发送 AppendEntries 请求，数据在 Follower 上没有冲突，则将数据暂时写在本地日志，Follower 的数据也还是 Uncommitted。Follower 将数据写到本地后，返回 OK。Leader 收到后成功返回，只要**收到的成功的返回数量超过半数** (包含Leader)，Leader 将数据 “sally” 的状态改成 Committed。( 这个时候 Leader 就可以返回给客户端了)  
Leader 再次给 Follower 发送 AppendEntries 请求，收到请求后，Follower 将本地日志里 Uncommitted 数据改成 Committed。这样就完成了一整个复制日志的过程，三个节点的数据是一致的，

除了上述的基本场景之外，还有许多其他场景下Raft的解决方案，此处只是将原博客中的基本情况进行整理，系统学习的话，需看原博主的博文：[共识算法：Raft](https://www.jianshu.com/p/8e4bbe7e276c)                                                                                                                                                                                                                                                                                                                                             
                                                             
## Zookeeper

### 简介

顾名思义 zookeeper 就是动物园管理员，他是用来管 hadoop（大象）、Hive(蜜蜂)、pig(小 猪)的管理员， Apache Hbase 和 Apache Solr 的分布式集群都用到了 zookeeper；Zookeeper: 是一个分布式的、开源的程序协调服务，是 hadoop 项目下的一个子项目。他提供的主要功 能包括：配置管理、名字服务、分布式锁、集群管理。

### ZooKeeper 的作用

#### 配置管理

在我们的应用中除了代码外，还有一些就是各种配置。比如数据库连接等。一般我们都 是使用配置文件的方式，在代码中引入这些配置文件。当我们只有一种配置，只有一台服务 器，并且不经常修改的时候，使用配置文件是一个很好的做法，但是如果我们配置非常多， 有很多服务器都需要这个配置，这时使用配置文件就不是个好主意了。这个时候往往需要寻 找一种集中管理配置的方法，我们在这个集中的地方修改了配置，所有对这个配置感兴趣的 都可以获得变更。Zookeeper 就是这种服务，它使用 Zab 这种一致性协议来提供一致性。现 在有很多开源项目使用 Zookeeper 来维护配置，比如在 HBase 中，客户端就是连接一个 Zookeeper，获得必要的 HBase 集群的配置信息，然后才可以进一步操作。还有在开源的消 息队列 Kafka 中，也使用 Zookeeper来维护broker的信息。在 Alibaba开源的 SOA 框架Dubbo 中也广泛的使用 Zookeeper 管理一些配置来实现服务治理。

#### 名字服务

名字服务这个就很好理解了。比如为了通过网络访问一个系统，我们得知道对方的 IP 地址，但是 IP 地址对人非常不友好，这个时候我们就需要使用域名来访问。但是计算机是 不能是域名的。怎么办呢？如果我们每台机器里都备有一份域名到 IP 地址的映射，这个倒 是能解决一部分问题，但是如果域名对应的 IP 发生变化了又该怎么办呢？于是我们有了 DNS 这个东西。我们只需要访问一个大家熟知的(known)的点，它就会告诉你这个域名对应 的 IP 是什么。在我们的应用中也会存在很多这类问题，特别是在我们的服务特别多的时候， 如果我们在本地保存服务的地址的时候将非常不方便，但是如果我们只需要访问一个大家都 熟知的访问点，这里提供统一的入口，那么维护起来将方便得多了。

#### 分布式锁

其实在第一篇文章中已经介绍了 Zookeeper 是一个分布式协调服务。这样我们就可以利 用 Zookeeper 来协调多个分布式进程之间的活动。比如在一个分布式环境中，为了提高可靠 性，我们的集群的每台服务器上都部署着同样的服务。但是，一件事情如果集群中的每个服 务器都进行的话，那相互之间就要协调，编程起来将非常复杂。而如果我们只让一个服务进 行操作，那又存在单点。通常还有一种做法就是使用分布式锁，在某个时刻只让一个服务去

干活，当这台服务出问题的时候锁释放，立即 fail over 到另外的服务。这在很多分布式系统 中都是这么做，这种设计有一个更好听的名字叫 Leader Election(leader 选举)。比如 HBase 的 Master 就是采用这种机制。但要注意的是分布式锁跟同一个进程的锁还是有区别的，所 以使用的时候要比同一个进程里的锁更谨慎的使用。

#### 集群管理

在分布式的集群中，经常会由于各种原因，比如硬件故障，软件故障，网络问题，有些 节点会进进出出。有新的节点加入进来，也有老的节点退出集群。这个时候，集群中其他机 器需要感知到这种变化，然后根据这种变化做出对应的决策。比如我们是一个分布式存储系 统，有一个中央控制节点负责存储的分配，当有新的存储进来的时候我们要根据现在集群目 前的状态来分配存储节点。这个时候我们就需要动态感知到集群目前的状态。还有，比如一 个分布式的 SOA 架构中，服务是一个集群提供的，当消费者访问某个服务时，就需要采用 某种机制发现现在有哪些节点可以提供该服务(这也称之为服务发现，比如 Alibaba 开源的 SOA 框架 Dubbo 就采用了 Zookeeper 作为服务发现的底层机制)。还有开源的 Kafka 队列就 采用了 Zookeeper 作为 Cosnumer 的上下线管理。

### Zookeeper 的存储结构

Znode：在 Zookeeper 中，znode 是一个跟 Unix 文件系统路径相似的节点，可以往这个节点存储 或获取数据。 Zookeeper 底层是一套数据结构。这个存储结构是一个树形结构，其上的每一个节点， 我们称之为“znode” zookeeper 中的数据是按照“树”结构进行存储的。而且 znode 节点还分为 4 中不同的类 型。 每一个 znode 默认能够存储 1MB 的数据（对于记录状态性质的数据来说，够了） 可以使用 zkCli 命令，登录到 zookeeper 上，并通过 ls、create、delete、get、set 等命令 操作这些 znode 节点

Znode节点类型：
1. PERSISTENT 持久化节点: 所谓持久节点，是指在节点创建后，就一直存在，直到 有删除操作来主动清除这个节点。否则不会因为创建该节点的客户端会话失效而消失。
2. PERSISTENT_SEQUENTIAL 持久顺序节点：这类节点的基本特性和上面的节点类 型是一致的。额外的特性是，在 ZK 中，每个父节点会为他的第一级子节点维护一份时序， 会记录每个子节点创建的先后顺序。基于这个特性，在创建子节点的时候，可以设置这个属 性，那么在创建节点过程中，ZK 会自动为给定节点名加上一个数字后缀，作为新的节点名。 这个数字后缀的范围是整型的最大值。 在创建节点的时候只需要传入节点 “/test_”，这样 之后，zookeeper 自动会给”test_”后面补充数字。
3. EPHEMERAL 临时节点：和持久节点不同的是，临时节点的生命周期和客户端会 话绑定。也就是说，如果客户端会话失效，那么这个节点就会自动被清除掉。注意，这里提 到的是会话失效，而非连接断开。另外，在临时节点下面不能创建子节点。 这里还要注意一件事，就是当你客户端会话失效后，所产生的节点也不是一下子就消失 了，也要过一段时间，大概是 10 秒以内，可以试一下，本机操作生成节点，在服务器端用 命令来查看当前的节点数目，你会发现客户端已经 stop，但是产生的节点还在。
4.  EPHEMERAL_SEQUENTIAL 临时自动编号节点：此节点是属于临时节点，不过带 有顺序，客户端会话结束节点就消失。

### Zookeeper提供的一致性服务

**顺序一致性：来自任意特定客户端的更新都会按其发送顺序被提交保持一致**。也就是说，如果一个客户端将Znode z的值更新为a，在之后的操作中，它又将z的值更新为b，则没有客户端能够在看到z的值是b之后再看到值a（如果没有其他对z的更新）。

**原子性：每个更新要么成功，要么失败**。这意味着如果一个更新失败，则不会有客户端会看到这个更新的结果。

**单一系统映像：一个客户端无论连接到哪一台服务器，它看到的都是同样的系统视图**。这意味着，如果一个客户端在同一个会话中连接到一台新的服务器，它所看到的系统状态不会比 在之前服务器上所看到的更老。当一台服务器出现故障，导致它的一个客户端需要尝试连接集合体中其他的服务器时，所有滞后于故障服务器的服务器都不会接受该 连接请求，除非这些服务器赶上故障服务器。

**持久性：一个更新一旦成功，其结果就会持久存在并且不会被撤销**。这表明更新不会受到服务器故障的影响。

**实时性**：在特定的一段时间内，客户端看到的系统需要被保证是实时的（在十几秒的时间里）。在此时间段内，任何系统的改变将被客户端看到，或者被客户端侦测到。
   
### 用CAP理论来分析ZooKeeper                                                                                                                

CAP这三个基本需求，最多只能同时满足其中的两项，因为P是必须的,因此往往选择就在CP或者AP中。

**在此ZooKeeper保证的是CP**

**分析：可用性（A:Available）**

**不能保证每次服务请求的可用性**。任何时刻对ZooKeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性；但是它不能保证每次服务请求的可用性（注：也就是在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要重新请求才能获得结果）。所以说，ZooKeeper不能保证服务可用性。

**进行leader选举时集群都是不可用**。在使用ZooKeeper获取服务列表时，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。所以说，ZooKeeper不能保证服务可用性。

