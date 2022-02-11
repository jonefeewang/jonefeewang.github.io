---
title: 一致性算法研究(三)Kafka
tags:
  - 分布式技术
  - kafka
  - zookeeper
  - consensus algorithm
  - zab
  - 一致性算法
categories: 分布式技术  
description: 一致性算法是分布式系统中的一个重要协议，是分布式数据库、Kafka、Zookeeper的基本协议。但这些协议晦涩难懂，对初学者需要耗费大量时间研究，本系列文章，是对一致性算法的一个大搜底和总览，通过对paper和技术资料的阅读，总结出一个简单易懂的学习材料，供其他人方便理解和学习。
date: 2020-02-26 21:36:47
toc: true
---
:warning: 原创文章，转载请注明出处
## [](#摘要 "摘要")摘要

一致性算法是分布式系统中的一个重要协议，是分布式数据库、Kafka、Zookeeper的基本协议。但这些协议晦涩难懂，对初学者需要耗费大量时间研究，本系列文章，是对一致性算法的一个大搜底和总览，通过对paper和技术资料的阅读，总结出一个简单易懂的学习材料，供其他人方便理解和学习。

## [](#关键词 "关键词")关键词

consensus algorithm(一致性算法) 分布式系统 一致性算法 Paxos Raft ZooKeeper ZAB

## [](#背景 "背景")背景

在谈到分布式系统时，很难不提及的产品除了ZooKeeper之外，另外一个就是Kafka. Apache Kafka是一个典型的分布式系统，整个系统从外围看，非常像我们说明共识问题时的例子 :

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%89-Kafka/replication.svg "distrabute replication")

S1-S5是Kafka整个集群，client 1 是生产端，client 2是消费端。整个集群支持消息的写入和读取，并且能保持不丢消息，不乱序。本节内容，我们一起来看下Kafka是怎么解决分布式场景下的共识问题的。

## [](#Kafka基本知识 "Kafka基本知识")Kafka基本知识

一个Kafka集群有若干台机器组成，一个集群可以运营成百上千个队列，一个队列可以有几个分区，比如下图所示，有3台机器(BrokerA、BrokerB、BrokerC)组成的一个集群，有两个topic： TopicA 和 TopicB。

TopicA有3个分区，分别标为重蓝色，重黄色，重粉色。3个分区平均分布在3台机器上，一台机器上一个。分区存储着队列的消息信息，在这个例子里，topicA有3个分区，消息也分3部分，分别存储在这3个分区上，但是Kafka集群为了规避单点风险，每个分区又会有一个或多个副本，副本数量可以自己定义，一般是1-3 个副本，1个副本的话是有单点风险的，2个副本容忍1个集群有1台机器宕机的情况下，队列仍然高可用，不影响消息的收发，3副本容忍2台机器同时宕机。所以，有f+1个副本的kafka topic容忍f个副本的机器同时宕机。

下图示例，topicA有3个分区，每个分区有3个副本，3个副本会平均分布在3台服务器上，比如topicA分区1主副本在BrokerA上，从副本rep1和rep2分别在BrokerB和BrokerC上，从副本会不断同步主副本的数据，和主副本保持一致，当主副本所在的机器宕机后，两个从副本所在的机器会在剩余的两个副本之间选出一个副本作为主副本，继续消息的收发，保持系统高可用。topicB有2个分区，只有一个副本。

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%89-Kafka/kafka.svg "kafka")

客户端有两个clientA 和 clientB。Kafka集群在接收客户端发送消息或读取消息的请求时，只会将请求发送给每个分区的主分区所在的服务器来处理。比如发送消息时，客户端将消息内容发送给topicA分区1的主分区所在的服务器BrokerA，由于分区1的副本rep1/rep2(replica 1和replica 2 下同)分别在BrokerB和BrokerC上，BrokerB和BrokerC上分别驻留一个消息复制器，负责把BrokerA上topicA主分区的消息拉给自身所在的从副本rep1和rep2上。

消息复制器在每台broker上都有一个，分别启动线程去其他服务器拉自己感兴趣的分区主本消息。

## [](#共识问题 "共识问题")共识问题

上边讲到topicA有3个分区，每个分区有3个副本，这样的队列容忍集群内同时有2台服务器宕机的情况下保持高可用。每个分区的3个副本平均分布在3台服务器上，那么问题来了，这3个副本怎么保持一致呢？

如下图所示，生产端在发送一条消息”id=101”后，消息发送到了分区的“主本“机器BrokerA上，消息的两个副本分别在Broker B和Broker C上，由上边所讲，Broker B和Broker C依赖本机的“消息复制器”将消息复制到本机副本中。如果”Client生产端“配置发送策略为”Ack=-1”，那么这条消息只有在Broker B和Broker C两台机器复制完成后，才算发送成功。这就保证了“消息如果发送成功”，那么其他副本一定是接收到消息了。

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%89-Kafka/msg_replicate.svg "kafka message replication")

但是，中间过程中的机器宕机等情况时，怎么来保证一致性呢？ 这个问题，kafka交给了zookeeper来解决。kafka集群启动时，会依靠zookeeper来选举一个集群的主节点服务器(controller)，controller来管理整个集群，比如队列的创建、删除，队列分区的选主，如前边所说的，topicA有3个分区，每个分区有3个副本(replica)，在创建每个分区时，kafka会使用zookeeper来选举一个分区副本的主(leader)。

一个集群内有上千个队列，每个队列都有数十个甚至数百个分区，每个分区又会有若干个副本，多个副本之间谁是leader副本？这些信息称之为metadata，kafka全部储存在了zookeeper上。

所以机器宕机时，controller和leader replica的选举任务都交给了zookeeper来完成，那么这时集群内机器之间的一致性，分区副本之间的一致性，这些共识问题都交给了zookeeper来解决。有了zookeeper，集群内每个机器不会对谁是controller再产生分歧，有了controller，一个集群就一个统一的“负责人”。同时，每个分区的副本之间使用zookeeper来选主，也会产生一个“负责人”，所有机器，多个副本之间都交由这些负责人来协调，不会再产生分歧，而且不会脑裂。

## [](#细节 "细节")细节

如下图所示，假设brokerA是当前集群的controller，当brokerA宕机后，其他broker马上会通过zookeeper来感知到，在brokerB/brokerC来选取一个新的controller出来。这个过程是通过抢占zookeeper的临时节点来实现的，因此共识问题、一致性问题都由zookeeper来保障。

同样切换的还有分区的leader，比如topicA分区1的leader原来在brokerA上，当brokerA宕机后，topicA分区1会在rep1和rep2之间选取一个新的分区leader，如下图所示，原来的分区1的rep1变成了新的分区leader，可以继续接受客户端的读写请求了。

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%89-Kafka/brokerA_down.svg "brokerA down")

在选取新的分区副本leader 时，kafka会优先选取消息数量和主本一样的副本，怎么实现的呢？

正常运行时，有3个副本的分区，kafka维持一个ISR(In sync replica)的概念，如果3个副本的消息量在一定时间内是一样的，那这三个副本就都在ISR内。如果某个副本的消息，在指定的时间内，没有同主本消息保持一致，这个副本就会被踢出ISR。如果3个副本里，主本一直接收消息，2个副本都在一定的时间内，没能跟上主本的消息数量，这时ISR缩减为主本一个，这时主本如果宕机会有丢消息的风险。因此，kafka有一个专门的设置，min ISR(in sync replica)数量，如果数量低于一定的值，kafka 集群就不再接收消息，因为这时集群是有丢消息的风险。这也是可用性(A)和一致性(C)之间的一个选择。

ISR的信息同样存储在zookeeper中，由zookeeper来保障信息的一致性。在客户端发送一条消息后，需要ISR里的各replica都确认收到消息后才能算成功，如下图所示:

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%89-Kafka/kafka_commit.svg "kafka commit msg")

与下图zookeeper的消息接收不一样，zookeeper只需要多数(quorum)确认收到即可：

![](/2020/02/26/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6-%E4%B8%89-Kafka/zk_commit.svg "zookeeper commit msg")

Kafka这中可以配置的确认机制，相比zookeeper会比较灵活一些，比如有9个机器组成的集群，zookeeper需要5个机器确认收到消息，而kafka只需要配置2或3即可保证消息不丢失，减少leader副本因等待接收ack而消耗的时间。

但在集群quorum比较小的情况，zookeeper的确认机制是比较占优势的。比如，当zookeeper的quorum是3的时候，如上图的情况，leader节点只需要收到任意一个follower的ack即可算发送成功，但是kafka需要等待两个副本都收到消息后，才能算发送成功。在这种场景下，zookeeper的确认机制是占优势的。

## [](#参考 "参考:")参考:

1.  [https://www.confluent.io/blog/distributed-consensus-reloaded-apache-zookeeper-and-replication-in-kafka/](https://www.confluent.io/blog/distributed-consensus-reloaded-apache-zookeeper-and-replication-in-kafka/)
2.  [https://www.semanticscholar.org/paper/Zab%3A-High-performance-broadcast-for-primary-backup-Junqueira-Reed/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3](https://www.semanticscholar.org/paper/Zab%3A-High-performance-broadcast-for-primary-backup-Junqueira-Reed/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3)