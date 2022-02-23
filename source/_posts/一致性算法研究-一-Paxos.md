---
title: 一致性算法研究(一)Paxos
tags:
  - 分布式技术
  - kafka
  - zookeeper
  - consensus algorithm
  - zab
  - 分布式协议
description: 一致性算法是分布式系统中的一个重要算法，是分布式数据库、Kafka、Zookeeper的基本协议。但这些协议晦涩难懂，对初学者需要耗费大量时间研究，本系列文章，是对一致性算法的一个大搜底和总览，通过对paper和技术资料的阅读，总结出一个简单易懂的学习材料，供其他人方便理解和学习。本章重点来研究下Paxos协议，理解Paxos算法的形成及场景分析。
toc: true            
date: 2020-02-26 21:35:30
categories: 分布式技术
---
 :warning: 原创文章，转载请注明出处
## [](#摘要 "摘要")摘要

一致性算法是分布式系统中的一个重要算法，是分布式数据库、Kafka、Zookeeper的基本协议。但这些协议晦涩难懂，对初学者需要耗费大量时间研究，本系列文章，是对一致性算法的一个大搜底和总览，通过对paper和技术资料的阅读，总结出一个简单易懂的学习材料，供其他人方便理解和学习。

## [](#关键词 "关键词")关键词

consensus algorithm(一致性算法) 分布式系统 分布式协议 Paxos Raft ZooKeeper ZAB

## [](#共识问题 "共识问题")共识问题

假如你有多个服务器，想让他们在某件事情上达成一致，这就是共识，共识意思是大家都对某事统一认可的。

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/consensus_problem.svg "图片信息描述")

如上图所示，有5个服务器，每个服务器上有一个processor，这些Processor要对一件事情达成共识。

共识问题在分布式系统中非常常见，比如多个服务器都有权限访问某个资源，怎么决定让哪个服务器去访问(互斥锁)？同等地位的多个服务器，哪个服务器去做master(leader选举问题)?多个服务器之间，怎么就一些事件的发生顺序达成一致性(有序复制问题)？

互斥锁和leader选举问题，大家经常使用zookeeper就很容易理解，比如通过抢占临时节点来选举leader,或获取互斥锁。

有序复制的问题，可以举个例子，比如有几台服务器组成的集群，这个集群提供读和写服务。客户端发送写请求，集群服务器能接受写的数据，并且自动复制数据到集群内的其他机器上，当某台机器宕机后，可以保持集群高可用，类似kafka集群.同时集群内每台服务器都可以提供读请求，以使整个集群提供高并发读服务。为了让集群正常工作，所有服务器必须将接收到的写请求能同步给其他服务器，实现所有服务器读一致性。

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/replicate_stat_ma.svg "复制状态机")

通常有两个方法达到这个目的：第一，集中所有写请求到其中的一台，这台机器作为协调器，他会保障所有的写请求按顺序的复制给其他服务器。

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/leader_cord.svg "leader协调")

第二，把所有的写请求发送给一个系统，这个系统协调所有的写请求落盘并且按顺序同步给其他服务器。

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/run_cons_p.svg "运行一致性算法")

第一种方法其实是通过共识算法选取一个leader，leader来处理和协调所有的写请求，第二种方法是运行一个共识算法系统，通过运行共识算法来确保所有机器按一定的顺序来记录客户端的写请求。

所以，共识算法用一句话来总结: 一个或多个系统提议一个或多个值(或议案)，怎么让所有的服务器对选取哪个值(或议案)达成一致。

## [](#一致性算法里的一些概念 "一致性算法里的一些概念")一致性算法里的一些概念

1.  议案、值(Value)
2.  提议(Propose)
3.  选取(choose/decide)
4.  进程(processor):代表共识算法里的一个参与方，实际意义是参与方服务器上的一个进程

## [](#一致性算法的运行环境 "一致性算法的运行环境")一致性算法的运行环境

1.  Processors
    1.  processor 运行速度不固定，有些运行快，有些运行慢
    2.  processor 有可能宕机
    3.  拥有存储的 processor 在宕机后，有可能携带状态再次加入协议运行
    4.  processor 不会欺骗、共谋或颠覆一致性算法
2.  网络
    1.  processor 可以向任何的processor发送消息
    2.  消息发送是异步的，可能需要一定的时间
    3.  消息可能丢失、乱序或重复
    4.  消息在传递过程中不会被破坏掉
3.  processor的数量
    1.  通常，一个共识算法正常运行，需要n=2F + 1个processor，才能容忍F个processor同时宕机

## [](#一致性算法的基本属性 "一致性算法的基本属性")一致性算法的基本属性

1.  Validity(有效性): 只有被提议的值才会被选取，不能选取一个未被提议的值.
2.  Uniform Agreement(一致性): 两个服务器不会选取不同的值
3.  Integrity: 每个服务器选取值的时候只会选取一次
4.  Termination: 所有的服务器最终都会选取一个值

## [](#一致性算法-paxos "一致性算法-paxos")一致性算法-paxos

paxos 就是一个一致性算法，用来解决上边所说的共识问题，通过运行paxos算法在一个分布式系统之间达成一致性。在这个系统内，客户端可以提交一个或多个值(value)，paxos算法促使整个系统最终选取一个大家都最终认可的值(value)。

paxos算法分为basic paxos和muti-paxos，一般大家说paxos都指的是basic paxos.Basic paxos运行一次只产生一个值(或议案)，如果需要重复运行的话，就会产生一连串的值，这就变成了muti-paxos.

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/paxos.svg "Paxos")

paxos将系统中的服务器分为三个角色,在实际运行中，一个processor有可能担任多个角色，者不影响协议的正确运行。

1.  proposers 议案提出者
    1.  整个协议的主动方，提出一个提案供大家通过
    2.  处理客户端(client)请求
2.  Acceptors 议案审议者
    1.  被动方：对proposer的提案做响应
    2.  对proposer的响应代表一次投票
    3.  存储被选中的议案
3.  Learners 议案学习接收者

## [](#paxos运行方式 "paxos运行方式")paxos运行方式

客户端(client)发起一个请求给任何一个proposer，proposer和acceptor运行一个两阶段协商，决定最终的选择值。paxos是多数派获胜原则，只要系统中超过50%的人同意，议案就会通过，这个原则保证系统不会出现脑裂的场景。通常，共识算法要求系统里有n=2F+1个参与者，这样的系统能满足F个参与者宕机。

在详细了解paxos算法之前，我们先来看下如果不使用paxos怎么来达成一致性:

**_在这个例子里，每个server既是proposer又是acceptor，具有双重角色。_**

### [](#1-使用一个acceptor "1.使用一个acceptor")1.使用一个acceptor

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/single_ac.svg "Single Acceptor")

1.  让一个机器来做选择，避免冲突和重复
2.  存在单点问题，如果机器宕机，则系统无法运行
3.  解决方法：使用多个acceptor(3,5..等奇数)，多数派原则（2/3，3/5，4/7），只要集群中多数机器选择了值v，那么v就确定为最终选择值,如果一个acceptor宕机，其他acceptor仍能记住这个值v，集群仍然能工作

### [](#改进1-增加多个acceptor "改进1: 增加多个acceptor")改进1: 增加多个acceptor

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/split_vote.svg "Split Vote")

1. 多个acceptor，每个server都接受第一个被提议的值，从而达成多数派原则

2.  产生脑裂，没有一个值是多数派选择的(3/5)，这意味着acceptor必要的时需要更改他所接受的值，而且需要经过多轮投票才能达成一致.（Accepted不是choosen，只有被多数派选择的才成为choosen）
    
3.  改进：Acceptor必须接受多个不同 的值，但最终值选定一个
    

### [](#改进2-Acceptor在多轮投票时，需要更改自己的值 "改进2: Acceptor在多轮投票时，需要更改自己的值")改进2: Acceptor在多轮投票时，需要更改自己的值

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/conf_choc.svg "Conflicting Choices")

1.  先后接受不同的值，最终会产生冲突
2.  加强：新的proposer(S5)提议之前必须看看有没有acceptor已经接受了其他server提交过议案，如果有的话，必须放弃自己的议案，提议原有的议案。(必须是个两阶段过程，第一阶段看是否有已经提的议案，第二阶段提自己的议案)

### [](#改进3-proposer提案时看看是否有其他议案已经被接受了 "改进3: proposer提案时看看是否有其他议案已经被接受了")改进3: proposer提案时看看是否有其他议案已经被接受了

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/conf_choc2.svg "Conflicting Choices")

1. 一旦整个系统(多数派)已经选中(chosen)一个议案，其他竞争的提议必须退出，S3已经接受了blue的提案，他必须拒绝"red"提案，

2.  提案必须有一个顺序，新的提案优于旧的提案，

### [](#改进4-提议必须有一个选后顺序，新的议案优先于旧的提议 "改进4: 提议必须有一个选后顺序，新的议案优先于旧的提议")改进4: 提议必须有一个选后顺序，新的议案优先于旧的提议

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/conf_choc3.svg "Conflicting Choices")

1.  给提议编一个唯一的提议遍号，编号大的提议具有优先权，acceptor在收到编号大的提议时，必须舍弃编号小的提议
2.  提案者每轮提的提议编号，必须是它所使用过、或见过的最大编号

### [](#改进4-继续 "改进4-继续")改进4-继续

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/proposal_num.svg "Proposal Number")

1.  怎么产生一个唯一的编号？使用（提案轮数+serverID)
2.  每个服务器都存储一个他所见到过的最大轮的议案编号
3.  每轮提议案时都将此编号增加一下，同时附加上本机的编号
4.  服务器宕机、或重启时，不能再使用原来的议案编号

### [](#7-总结Paxos是一个两阶段协议 "7.总结Paxos是一个两阶段协议")7.总结Paxos是一个两阶段协议

1.  第一阶段: 广播Prepare请求(RPC请求)
    1.  找到有没有已经被选择(Chosen)的值
    2.  停止老的还未被Chosen的议案
2.  第二阶段:广播Accept提案请求(RPC请求)
    1.  广播一个议案，让Acceptor接受一个值，让多数派通过这个议案，达到最终选定一个议案

### [](#一张图看完整个Paxos运行过程 "一张图看完整个Paxos运行过程")一张图看完整个Paxos运行过程

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/paxos_one_page.svg "Whole Paxos Process In One Page")

## [](#Basic-Paxos-分析-提案之间的竞争 "Basic Paxos 分析: 提案之间的竞争")Basic Paxos 分析: 提案之间的竞争

### [](#场景一-老提案已经通过，新提案会找到它，并接收它 "场景一: 老提案已经通过，新提案会找到它，并接收它")场景一: 老提案已经通过，新提案会找到它，并接收它

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/prev_chose.svg "Proposal Number")

### [](#场景二-老提案还未完全被接受，新提案看到了，并开始使用它，两个proposer都会成功 "场景二: 老提案还未完全被接受，新提案看到了，并开始使用它，两个proposer都会成功")场景二: 老提案还未完全被接受，新提案看到了，并开始使用它，两个proposer都会成功

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/new_see.svg "Proposal Number")

### [](#场景三-老提案还未完全被接受，新提案没找到它：老提案被block，新proposer用它自己的议案 "场景三: 老提案还未完全被接受，新提案没找到它：老提案被block，新proposer用它自己的议案")场景三: 老提案还未完全被接受，新提案没找到它：老提案被block，新proposer用它自己的议案

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/lost_prev.svg "Proposal Number")

### [](#场景四-两个竞争型的议案，甚至可以产生活锁 "场景四: 两个竞争型的议案，甚至可以产生活锁")场景四: 两个竞争型的议案，甚至可以产生活锁

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/live_lock.svg "Proposal Number")

解决方法：1. proposer在重新提案之前，可以等待一个随机时间，给其他proposer完成accept的机会2. Muti-Paxos可以选择一个leader，省去第一阶段，直接进入accept阶段

## [](#Basic-Paxos-其他要点 "Basic Paxos 其他要点")Basic Paxos 其他要点

1.  只有提案者(proposer)知道哪个议案通过了
2.  如果其他的server需要知道，他们必须自己提一个议案来运行一次paxos，才能知道

## [](#Basic-Paxos-容灾 "Basic Paxos 容灾")Basic Paxos 容灾

Proposer、Acceptor在不同的时间宕机  

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%80-Paxos/ha.svg "Proposal Number")

1.  Proposer在提案之前宕机了
    1.  任何事情也没做，相当于什么事情也没发生
2.  Proposer在Prepare之后宕机了
    1.  acceptor接收到prepare消息，没有收到后续的accept请求，其他的proposer可以运行更高编号的提案继续下去
    2.  如果proposer在acceptor接收accept的过程中宕机了，那么只要有一个acceptor接收到了，后续的proposer看到后，会将这个提案继续下去，参照上边的场景二。如果没看到，就会miss掉，老的提案被block，新提案获得通过，参照场景三。
3.  Acceptor在Accept之前宕机了  
    只要集群内存在多数派的accetpor，整个系统就能运行下去
4.  Accetpor在Accept之后宕机了  
    只要集群内存在多数派的accetpor，整个系统就能运行下去

## [](#参考-Basic-Paxos-详细运行过程 "参考: Basic Paxos 详细运行过程")参考: Basic Paxos 详细运行过程

完整的协议运行过程，可以总结为两阶段和3p1a：

### [](#一阶段a-Proposer-PREPARE "一阶段a: Proposer (PREPARE)")一阶段a: Proposer (PREPARE)

proposer 发出一个prepare消息，消息包含一个值，这个值对于这个proposer来说是唯一的,不重复的，每次prepare可以递增来保证唯一
```
    ID = cnt++;  
    send PREPARE(ID)  
```
### [](#一阶段b-Acceptor-PROMISE "一阶段b: Acceptor (PROMISE)")一阶段b: Acceptor (PROMISE)

acceptor在收到prepare message后的处理逻辑
``` 
if (ID <= max_id)  
 do not respond (or respond with a "fail" message)  
else  
 max_id = ID     // 存储目前我见到的最大的ID  
 if (proposal_accepted == true) // 是否之前已经接受了某个值？  
 respond: PROMISE(ID, accepted_ID, accepted_VALUE)  
 else  
 respond: PROMISE(ID)  
```
### [](#二阶段a-Proposer-PROPOSE "二阶段a: Proposer (PROPOSE)")二阶段a: Proposer (PROPOSE)

Proposer检查所有acceptor的响应，检查之前是否有已经被接受(accepted)的值，
我(Proposer)是否接收到多数acceptor的promise响应？

```
 if yes  
 do any responses contain accepted values (from other proposals)?  
 if yes  
 val = accepted_VALUE    // 改变议案为某个acceptor之前已经接受的议案  
 if no  
 val = VALUE     // 可以使用原有的议案  
 send PROPOSE(ID, val) to at least a majority of acceptors  
```
### [](#二阶段b-Acceptor-ACCEPT "二阶段b: Acceptor (ACCEPT)")二阶段b: Acceptor (ACCEPT)

每个acceptor接收到一个从Proposer发来的 PROPOSE(ID, VALUE)请求，如果这个请求的ID是我见过或处理过的最大ID，那么我就接受这个值(议案)，同时返回这个议案给proposer和所有的learner.
 
```
if (ID >= max_id)  // 这是我见过的最大ID么?  
 proposal_accepted = true     // 记录下来，我们接受了一个议案  
 accepted_ID = ID             // 保存提案编号  
 accepted_VALUE = VALUE       // 保存议案  
 respond: ACCEPTED(ID, VALUE) to the proposer and all learners  
else  
 do not respond (or respond with a "fail" message)  
```
如果大多数acceptor都接受了这个值(议案)，那么最终就达成了一致性，选取这个值(议案)

## [](#参考 "参考")参考

1.  [Diego Ongaro，2013，Paxos Lecture](https://www.youtube.com/watch?v=JEpsBg0AO6o&t=1931s)
2.  Paul Krzyzanowski, 2018, [https://www.cs.rutgers.edu/~pxk/417/notes/paxos.html](https://www.cs.rutgers.edu/~pxk/417/notes/paxos.html)
3.  Leslie Lamport, 2011, Paxos Made Simple