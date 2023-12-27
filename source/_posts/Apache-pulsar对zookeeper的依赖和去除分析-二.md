layout: _post
title: Apache pulsar对zookeeper的依赖和去除分析(二)
date: 2023-09-15 12:13:00
description: 本文分析了apache bookkeeper对zk的依赖和去除思考
tags:
  - Apache Pulsar
  - Apache Bookkeeper
  - Apache Kafka
categories: 消息队列
toc: true
---

# 背景
Apache pulsar是近年来国内比较时髦的消息队列，而且是开源的产品，有不少国内的互联网公司都开始了使用了。针对开源的几个消息队列产品比较，可以参看我之前的一篇博客[<<消息队列业界调研>>](/2021/03/26/消息队列业界调研/ "消息队列业界调研")。Pulsar为什么现在比较火，说白了是搭上了"云原生"这趟车。在当今时代，任何互联网技术，特别是基础组件，如果没云原生化，出门别人都不好意思给你打招呼。特别是从2021年开始， 突然一切互联网技术都需要"云原生"了，其中隐情也是各有使然，当然这是另一个话题。(关于我在美团时负责的消息队列Mafka云原生化分析，可以参考我的这篇博客和 [<<消息队列Mafka列长期发展计划>>](/2021/07/14/Mafka-LRP/ "Mafka消息队列长期发展计划") 和 [<<消息队列Mafka全链路弹性伸缩演进策略>>](/2021/07/22/Mafka全链路弹性伸缩演进策略/ "Mafka全链路弹性伸缩演进策略")  )Pulsar因为其"天然"的存算分离架构，和云原生讲究的弹性伸缩性(scalability)，品性特别符合，自然受到了大家特别的追捧。

关于pulsar架构的评价，可以参考我之前写的这篇博客[<<消息队列业界调研>>](/2021/03/26/消息队列业界调研/ "消息队列业界调研")。本系列文章主要分析下pulsar对zookeeper的依赖，因为在当前的架构下(本篇以pulsar 2.8版本分析)，pulsar对zookeeper是强依赖关系，说白了就是zookeeper一旦挂掉，pulsar整个集群也就挂掉了。因为之前我也有一篇博客分析kafka 去除zookeeper依赖[<<揭秘Kafka去Zookeeper技术>>](/2021/12/01/Kafka去Zookeeper揭秘/ "Kafka去Zookeeper揭秘")，本篇分析下bookkeeper对zookeeper的依赖，以及如何去除。因为涉及的内容比较多，会分为三篇博客来写，第一篇分析pulsar本身，即pulsar broker，第二篇分析 bookkeeper对zookeeper的依赖，就是本篇内容，第三篇 是临时加的，因为2023年5月份，pulsar官方出了一个类似kafka kraft的一致性组件([moving-toward-zookeeper-less-apache-pulsar][1])来去除pulsar对zookeeper的依赖，第三篇会重点分析下这个组件。

[1]: https://streamnative.io/blog/moving-toward-zookeeper-less-apache-pulsar "moving-toward-zookeeper-less-apache-pulsar"

# 架构简介

## 概览

在讨论细节之前先来重温下pulsar的架构，pulsar架构的详细介绍，可以参考我之前的博客[<<消息队列业界调研>>](/2021/03/26/消息队列业界调研/ "消息队列业界调研")一文
![](pic5.svg)
1. 整个pulsar 分为broker和bookkeeper两大部分
2. broker 和 bookkeeper都依赖 zookeeper来做元数据存储
3. broker是无状态的计算层(有部分配置缓存和消息数据缓存)
4. bookkeeper是存储层, bookkeeper实际上是apache的另一个开源项目

## 先来看几个几个名词概念
1. ledger    
     ledger是bookkeeper中概念，代表一个流水账，一个ledger其实就是 pulsar中topic的一个partition。但是在bookkeeper内部实现里，ledger其实是一个虚拟的概念，没有物理实体或文件实体。它有多个sgement组成，因为bookkeeper把多个leger写入到同一个entry log内，不同的legder会相互间隔相连，属于同一个legder的entry会排列在一起。entry log会不断滚动，生产一段一段的文件，每一段称为一个sgement。
2. journal
      bookkeeper中的物理文件，journal是短暂存储.在接收到客户端的发送的消息后，先写入write cache，然后再异步落盘形成entry log, entryFile是长期存储的消息文件, 最后在写入journal，在经过group commit方式flush到磁盘落地后，再ack客户端发送消息成功。jornal只是为了恢复entry file而写的。
3. entry log 
    entry log 是bookkeeper中的物理文件，是消息数据的持久存储。可以在bookkeeper内配置entry log的目录，可以是一个或多个，每个目录只对应一个entry log，所有ledger都落盘到这个file内。这个跟journal比较像，都是所有ledger的消息数据，但是entry log里的数据是按照ledger排列的，方便读取，而且entry log 是异步刷盘，journal是raw的消息数据，是同步刷盘(group commit方式)。当bookkeeper机器宕机后，未落盘的entry log 数据，可以根据check point从journal内恢复，这就是journal作为短暂临时存储的目的。
4. index file 
    bookkeeper的存储是有几种实现(接口称为LedgerStorage)可选:     
      - InterleavedLedgerStorage
      - SortedLedgerStorage
      - DbLedgerStorage
    
    前两种实现使用index file物理文件来作为entry的索引查找，最后一种使用rocksDB来做索引查找. 主要是查找某个特定entry的位置，比如给一个entry ID和一个ledger ID，查找这个entry在哪个entry log file里,在文件内的offset是多少。
5. fragment
    在发送消息时，如果ensemble里有一个bookkeeper机器掉线，客户端会再选择一个ensemble，继续发送消息，每个ensemble对应的一批消息称为fragment 
6. ensemble(EM)
    几个bookkeeper实例组成一组存储，比如整个集群有10台机器，选择其中5台作为一个ensemble，WQ设置为3，AQ设置为2.
7. WQ
    write quorum 发送消息时，必须写入的bookkeeper机器数量。在实际项目实践中，WQ一般和ensemble设置为一致，防止产生striping(一部分entry在一台机器上，下一部分跳到了另外一批机器上)而降低读消息的效率    
8. AQ
    ack quorum 发送消息时，必须收到ack的bookkeeper机器数量   
9. Fencing
    当第一个客户端掉线后，第二个客户端必须关闭第一个客户端发送消息时写入的ledger，在关闭之前防止之前的ledger再继续写入，会先做fence操作，防止之前的客户端继续写入，产生脑裂，因为bookkeeper中的一个ledger写入，不允许有两个客户端同时写入，客户端必须保障这一点。

# 概述
bookkeeper对客户端有一定的要求:

- bookkeeper要求客户端在写入ledger的时候，不允许两个客户端同时写入，客户端必须保障这一点
- bookkeeper要求客户端在写入ledger的时候，ledger Id必须是顺序的，客户端必须保障这一点

相对于pulsar整体架构来说，pulsar的broker就是bookkeeper的客户端，所以broker会强依赖zk，保障任何时候一个topic只能被一个broker所负责。bookkeeper本身是一个无脑的存储，上边所说的两个保障都需要客户端自己来做。

数据的副本，也是靠客户端来保障的，比如像EM=3, WQ=3,AC=2这样集群设定，一个ensemble有3台bookkeeper机器，write quorum是3，Ack quorum是2，那客户端在发送消息时，必须发送3次，保障每台机器都发送到，但是可以在收到2个bookkeeper的ack后就算成功。

所以，bookkeepe本身对zk的依赖来说不重，主要做集群内机器上下线通知，和一些元数据存储，下边来详细看下zk上的元数据。

# Broker对Zookeeper的依赖总结

| 项目                                            | zk 路径                                               | value/children  | 备注                                                                                         |   | 
|-------------------------------------------------|-------------------------------------------------------|-----------------|----------------------------------------------------------------------------------------------|---|
| 集群节点相关 | /ledgers/available/[ip:port]                                       | children        | 临时节点，当前在线的集群节点列表。客户端会来读取，创建新的ensemble                                                                                     |   |
|                                                 | /ledgers/cookie/[ip:port]                             | children        |  持久节点，broker启动时自动注册，用来核实bookkeeper节点整个生命周期的配置信息|   |
|                                                 | /ldegers/available/readonly/[ip:port]                        | children           | 启动时强制注册为readonly的节点，或者当磁盘无可用空间时转换为readonly，这样的节点无法接受写操作                                                                 |   |
| ledger数据相关的                                | /ledgers/00/0000/L000X                | value           | ledger的metadata信息，例如每个fragment的ledger id范围，方便ledger读取时，确定ledger在哪台机器上    |   |
|                                                 | /ledgers/idgen/ID-0000000000X                             |   value | 每次创建新ledger时，从这个节点获取一全局唯一的ledgerId               |   |

从上边的列表，不难看出，zk在这里的作用分两类，第一个是集群节点上下线注册和通知，第二类是全局唯一的锁，比如ledger Id的生成，以及ledger metadata的写入和更新。第三类，完全是配置类型的存储。

第一类，节点在zk上的注册，如果有网络抖动的话，或者zk自身jvm gc会造成节点瞬间下线，如果抖动时间比较长，会影响单个节点，或整个集群的稳定性，特别是zk数据量大了以后，自身jvm gc造成长时间卡住或假死，会造成全部节点下线，可用性降为0。所以如果条件允许，作为节点注册上下线，探活一类的zk，一定不能存储太多的数据。如果ledger自身数量比较多，考虑拆分集群，或替换为持久节点，外部增加一个scanner组件来主动扫描有问题的节点，如果节点端口不响应，可以将持久节点状态置为dead状态，或删除(在美团时，美团消息队列Mafka就是这样做的)。

第二类，全局唯一锁，这个可以使用redis做一个简单全局锁，或其他分布式锁组件，以此来替换掉zk。

第三类，配置存储类的，可以迁移到kv或mysql一类的关系数据库里，避免zk数据量的线性增长。