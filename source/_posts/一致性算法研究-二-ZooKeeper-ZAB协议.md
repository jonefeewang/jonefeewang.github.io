---
title: 一致性算法研究(二)ZooKeeper ZAB协议
toc: true
tags:
  - 分布式技术
  - kafka
  - zookeeper
  - consensus algorithm
  - zab
  - 分布式协议
description: 
        一致性算法是分布式系统中的一个重要算法，是分布式数据库、Kafka、Zookeeper的基本协议。但这些协议晦涩难懂，对初学者需要耗费大量时间研究，本系列文章，是对一致性算法的一个大 搜底和总览，通过对paper和技术资料的阅读，总结出一个简单易懂的学习材料，供其他人方便理解和学习。            
date: 2020-02-26 21:36:37
categories: 分布式技术
---

:warning: 原创文章，转载请注明出处

## [](#摘要 "摘要")摘要

一致性算法是分布式系统中的一个重要算法，是分布式数据库、Kafka、Zookeeper的基本协议。但这些协议晦涩难懂，对初学者需要耗费大量时间研究，本系列文章，是对一致性算法的一个大搜底和总览，通过对paper和技术资料的阅读，总结出一个简单易懂的学习材料，供其他人方便理解和学习。

## [](#关键词 "关键词")关键词

consensus algorithm(一致性算法) 分布式系统 分布式协议 Paxos Raft ZooKeeper ZAB

## [](#背景 "背景")背景

Zookeeper是大家工作中经常用到的分布式组件，用来处理分布式场景下一致性问题。按理说Zookeeper使用paxos来解决共识问题，是顺利成章的，但zookeeper确是借鉴了Paxos协议，发明了自己的ZAB(Zookeeper Atomic Broadcast protocal简称ZAB)协议来解决共识问题。Paxos、ZAB、Raft这个几个协议都认为自己是一个独立的一致性算法，但有人却说，世界上只有一个一致性算法，那就是Paxos，其他协议都是它的变种。这种理论争论暂且不提，我们来研究下具体协议的真实工程实践，毕竟Zookeeper在实际工程实践中，是一个成功的案例。

## [](#问题原型 "问题原型")问题原型

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/consensus_problem.svg "Consensus Problem")

在上一篇Paxos研究中，提到了共识问题。其中一个例子，讲在一个分布式系统中，client端往系统写数据，server端接受数据，同时提供读服务。为了让系统的吞吐量高，希望所有的节点都提供写和读服务。要达到这样的效果，显然需要数据在集群内各节点做复制，否则无法读到。要解决这样的分布式数据复制，有个哥么写了一篇论文，说有两种方法：第一使用一个replication state machine(复制型的有限状态机)。第二种方法，使用一个primary-backup 主备系统。Zookeeper也是一个分布式系统，问题场景和上边的图形很相似，但Zookeeper说自己属于第二种方法，Paxos属于第一种。这个问题，我们暂时先知道即可，关于具体的复制有限状态机(replication state machine)和主备系统(primary-backup)系统的区别，我会在以后的纯理论研究中去分析。下边我们来研究zookeeper怎么来具体解决共识问题。

## [](#Paxos实际应用中的缺陷 "Paxos实际应用中的缺陷")Paxos实际应用中的缺陷

Paxos只确只要有足够的节点在工作，系统最终能选出一个议案，但有以下缺陷：

1.  不保序
    
2.  容忍丢失消息
    

例如在以下的特定场景中，有3个独立的Proposer: P1,P2,P3，三个acceptor，P1,P2运行过程中，先后宕机。  

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/paxos_problem.svg "Paxos的问题")

  
通过上图可以看出

1.  客户端提案的顺序按编号应该是27-A，28-B，29-C，但最终应用后确是27-C，28-B，29-D，
    
2.  提案A丢失。
    

## [](#Zookeeper基础知识 "Zookeeper基础知识")Zookeeper基础知识

Zookeeper解决了上边所说的两个问题，他使用了Primary-backup的方案来保障所有的server状态一致。在这样的系统下，所有的客户端请求都转发给一个server，这个server称为primary，在处理完这个请求后，将结果广播给其他所有的server，这个广播协议称为ZAB()。在primary宕机后，其他server会执行一个recovery的过程，选举一个新的primary出来，在新primary履职前，必须统一所有server的状态，让大家对当前的数据状态保持一致。  

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/zab.svg "ZAB")

### [](#1-Requests-Transactions-Identifiers-zxid "1: Requests,Transactions,Identifiers(zxid)")1: Requests,Transactions,Identifiers(zxid)

zookeeper中exists、getData、getChildren等读操作，都是本地处理的，无论是leader或follower或observer都可以就地处理，所以读操作都是比较快的。

create/delete/setData等都是写操作，写操作都会forward给leader处理，相比读操作会较慢。每个写操作都称为一个transaction，client发起一个事务(写操作)，zookeeper会将事务转发给zookeeper leader来处理。

每个transaction包含两个值: 新value和一个version，比如<1,1>，客户端发起这个事务，如果最终被提交的话，server端会替换节点为新的value和version，而不是去增加原value的值。

每个事务都是原子操作，value和version都会被更新，不会出现一个更新了，而另外一个没更新。

所有的事务都是幂等的，可以执行一次或多次，但前提是每次执行都是按照相同的顺序执行。

当leadrer生成一个事务时，会产生一个transaction ID，叫做zxid，zxid标识唯一一个事务。zxid（64位）包含两部分epoch(位) + counter（32位），epoch部分是一个server 履职leader期间的一个标识，counter部分是递增的，每收到一个事务就会加1。leader广播这样的事务，其他follower来接收。这样有一个好处，就是当leader宕机后，新leader选举时可以比较各server接收到的zxid，看谁的zxid更大,这样选举时就知道谁接收的transaction比较多了。 。

### [](#2：Zk集群leader选举 "2：Zk集群leader选举")2：Zk集群leader选举

leader是一个集群(ensemble: Leader+Follower)选举出来的一个主，负责协调整个集群的事物. leader会处理改变集群状态的操作(create/deate/setData)，把每个操作转换成一个transaction，发起proposel给follower，accept后按顺序commit。

要选举一个leader，一个集群必须有法定人数(quorum:超过一半的人数)的节点支持他，而且quorum数量必须是奇数，否则会产生脑裂。

一个刚启动的节点状态处于Looking状态，他必须选取一个leader或寻找一个已存在。如果没有leader，集群会选择一个leader，被选出的leader会处于leading状态，其他结点会处于following状态。

选举leader时，使用的选举协议比较简单，所有节点都会发出notification消息，notification消息包含server id (sid)和最近一次commit的zxid(实际上是epoch和一个counter).

每个server在选举时，会执行以下步骤:

1.  假如voteId，voteZxid是当前节点收到的选举消息，myZxid和 mySid是当前节点自己的值
    
2.  如果voteZxid > myZxid 或 voteZxid = myZxid，但是voteId > mySid，把当前收到的voteZxid和voteId保留。
    
3.  否则把自己的myVoteid和myZxid赋值给VoteZxid和VoteId。
    
    简单来讲，拥有最大zxid的节点会赢得选举，如果大家都拥有最大zxid，服务器sid最大的获选
    

一旦某个server接收到了达到法定数量(quorum)的选举，他就声明获选。获选的server会行使leader角色，否则他变成follower，连接当前的leader节点，一旦连接上后，必须接收完来自leader的同步信息，他才能开始处理请求

1.  下边展示了一个“正常“的leader选举过程：
    
    ![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/leader_elect.svg "Zookeeper Leader选举")
    
2.  下边这个因为网络延迟，造成的一个“非正常”选举过程
    
    ![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/msg_delay.svg "Zookeeper 选举中的消息延迟")
    
3.  如果S2稍微等待长一些的话，就可以一次选成功，省去中间的“无用功”  
    zookeeper设置了一个固定长度的时间200ms，节点必须等待这么长时间才能进行选举，这个时间远远大于正常情况下的网络之间的延迟

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/elect_delay.svg "延迟等待")

## [](#Zookeeper中事务的提交 "Zookeeper中事务的提交")Zookeeper中事务的提交

### [](#基本流程 "基本流程")基本流程

Follower在接受到写请求后，会将请求转发给集群leader，leader会将请求转换成事务，在集群内进行提交和广播，当然这需要一个协调过程。过程中，leader和follower之间的交互，使用的是zab协议。

假设现在有一个正常的集群，集群有一个leader，ZAB提交一个事务的过程如下:

1.  leader发送一个proposal消息p，给所有的followers
2.  收到消息p后，follower返回leader一个ack，告诉leader，他已经接受了这个协议
3.  如果leader收到了仲裁数量的ack，则通知followers提交commit这个事务

如下图所示:

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/zk_transaction.svg "ZooKeeper中的事务")

第二步时，follower需要检查这个proposal是否来自于leader，而且ack和commit事务必须按照leader广播的顺序。

### [](#事务日志和快照 "事务日志和快照")事务日志和快照

Zookeeper在运行中，会生成两个文件，一个是事务log文件，一个是snapShot快照文件。事务log文件是zookeeper在接收事务的proposal时，记录在磁盘上的append only文件，snapShot文件是zookeeper内存数据库的快照文件。snapShot文件会频繁生成，但是server是一直运行状态的，而且在生成快照时，还会有新的事务进来，所以在某一刻，快照和zookeeper的内存数据库并不是完全一样的。不过这没关系，在生成快照时，zookeeper会记录快照开始前的最后一个zxid，当zookeeper server重启时，为了恢复内存数据库，他会载入snapShot时，重新播放(replay)事务log文件来补充快照生成中或生成后，没有记录下的事务。

为了保障数据的一致性，zookeeper要求follower在接收proposal后必须真实落盘(不能只是写入磁盘缓存disk cache)才能ack，但一个事务一次写盘，显然是不高效的，因此zookeepr在写磁盘时使用了group commit和padding的方式写盘，多条事务只有一次写盘操作。

## [](#ZAB协议对消息顺序和消息不丢失的保障 "ZAB协议对消息顺序和消息不丢失的保障")ZAB协议对消息顺序和消息不丢失的保障

ZAB 保障了一些重要的原则:

1.  如果leader广播事务时是先T后T’，那么每个server必须先commit T后再commit T’.
    
2.  如果有一个server提交事务是按照T/T’顺序,那么其他所有的server必须先提交T，后提交T’.
    

**第一条原则保障了事务是按相同的顺序投递到其他server的(保序)，第二条原则保障了任一server不跳过事务(不丢失消息)。**

Zookeeper使用Tcp协议在Server之间通信，保证了消息的严格有序(FIFO)。同时，所有的请求都会按顺序先发给Leader，Leader使用两阶段提交方式来处理(先Propose和后Commit)，在commit之前，必须拥有足够仲裁数量的机器先接受消息，在接收消息后，follower会将消息存储在磁盘上(以zxid形式append Log)，这保障了事务的严格有序。

在实现ZAB广播协议里，最难的是处理双leader，比如老的leader发送的心跳延迟或丢失，都会造成follower认为leader已经宕机，促使他们选举一个新的leader。双leader会造成server提交事务时顺序发生变化，或者跳过事务。

要解决这个问题，ZAB保障:

1.  一个获选的leader一定要commit所有最终会被提交的事务，在广播新时代事务之前。
2.  任何时候，不会有两个server都拥有仲裁数量的支持者

为了保障第一条，一个当选的leader不会行使职权，直到仲裁数量server对他的初始状态达成一致，这个过程称之为同步(Synchronization)。 初始状态必须包含所有之前已经提交的事务，而且也包含之前最终会被commit但是还未commit的事务。为了保障第二条，是使用了一个技巧，就是两次形成的仲裁必定都包含同一个server。下图说明了这两点：

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/leader_trans.svg "leader切换")

1.  Server 3就是两次仲裁的交叉Server，原集群有他，新的leader的集群也有他
2.  发生切主前，形成仲裁数量的Server是Server(1-5)。发生切主后仲裁数量的server是1-3(3>5/2)，他们支持Server 3，但Server 5已经没有足够数量的仲裁Server支持他了，只有Server 4和他本身。
3.  在Server 1-3之间发生leader选举时，因为Server 3多接受了一个事务(1,1)，所以的他的 zxid会比其他两个Server大，所以他必定会别选取为Leader。Server 3被Server 2和Server1 选取为leader后，形成一个法定人数仲裁群(quorum)，之后新的仲裁群会忽略掉原集群leader的所有消息。
4.  事务<1,1>被Server 3和Server 4接收到，并返回了Ack，和Server5 自身ack自己，促成了一个多数(quorum)Ack，所以事务<1,1>最终应该被提交。在新的Leader集群里，事务<1,1>在Server 3上有记录，server 3会最终提交这个事务。
5.  如果事务<1,1>在leader(Server 5)宕机前，proposal没有广播到达具有法定人数的servers上，那么这个条事务有可能最终被提交，也有可能被丢失，取决于广播到的server是否会参与新的leader选举。但这条事务因为没有达到法定人数的ack，所以leader还是返回给客户端请求处理失败的，防止客户端丢失这条消息。

在时代交替时，如果follower和leader的事务差异不大，leader是可以发差异化事务包给follower补齐。如果差异很大，那么leader就会发整个snapShot给follower，这会加大集群恢复的时间。

## [](#总结 "总结")总结

### [](#ZAB优势 "ZAB优势")ZAB优势

从整个zookeeper全局来看，ZAB保证了以下2个特性:

1.  可靠的消息投递  
    如果一个消息被投递到一个server，那么其他server最终也将会有这条消息。相比Paxos，消息被投递到一个Server后，因为系统之间延时等问题，有可能丢失掉。
2.  完全有序  
    如果client投递a,b两条消息按a先于b的顺序，那么所有server收到的消息顺序同样也是a先于b.

依靠这两个特性,zookeeper能保障整个集群中状态的一致性。

### [](#ZAB问题 "ZAB问题")ZAB问题

1.  从客户端看到的不一致问题  
    因为在广播commit的阶段，leader和各follower之间有网络延迟，到达follower时先后时间点是不一样的，这就注定了有些follower是比其他更快的读取到最新的数据。
2.  一个client两个链接，造成的消息不守序  
    在一个client端建立一条跟zookeeper的链接时，消息按严格有序的方式投递到leader，并且按照严格的顺序被广播给各follower。但如果一个客户端机器建立两个链接时，发送给两个链接的消息是不能保序的。
3.  重复消息投递的可能性  
    在上边ZAB保序的讨论例子里，如果事务<1,1>在leader宕机前没有被commit，而且接收到proposal的follower也不确定是哪几台，那么新的leader选举时，包含新proposal日志的server不一定参与集群选举，如果参与的话，该条日志就会最终被zookeeper集群commit，如果没有参与，那么这条事务就丢失了。虽然这两种情况，都是在client端确定本次请求失败时发生的，但前者在server端该条事务确实被记录下来了，如果client补发他认为“失败”的这条事务，那么这条请求是被重复投递了。

但在zookeeper集群内部来看，通过使用ZAB，能保障所有的客户端能观察到所有的数据更新事件，每个server都不会遗漏一个事件，而且是按一致的顺序来执行。但不一定是在同一时刻都看到相同的事件执行状态(因为上边所说的第一条原因)。

## [](#ZooKeeper的吞吐量测试 "ZooKeeper的吞吐量测试")ZooKeeper的吞吐量测试

下图是一个性能测试，消息大小是1024个字节，竖轴是吞吐量，横轴是一个zookeeper集群的个数。  
机器配置: 双cpu，4核处理器 Xeon 2.5GH，16G RAM, 1T SATA disk.

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%BA%8C-ZooKeeper-ZAB%E5%8D%8F%E8%AE%AE/benchmark.png "Zookeeper性能测试")

### [](#图中曲线说明 "图中曲线说明:")图中曲线说明:

1.  Net only: 不落盘，只有网络处理
2.  Net+Disk: 网络请求处理，增加落盘写事务log文件
3.  Net+Disk(no write cache): 网络请求处理
4.  Net cap: 理论纯网卡测试网络吞吐量

从图中可以看到：

1.  在disk cache被关闭后，zookeeper变成IO bound了
2.  随着集群机器数量的增加，ops也开始降低，因为受限于leader的网卡瓶颈，leader需要复制数据到其他机器(follower或observer)。

参考:

1.  ZooKeeper: Distributed Process Coordination /Flavio Junqueira and Benjamin Reed 2013
2.  [https://www.semanticscholar.org/paper/Zab%3A-High-performance-broadcast-for-primary-backup-Junqueira-Reed/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3](https://www.semanticscholar.org/paper/Zab%3A-High-performance-broadcast-for-primary-backup-Junqueira-Reed/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3)
3.  [https://dl.acm.org/doi/10.1145/1529974.1529978](https://dl.acm.org/doi/10.1145/1529974.1529978)