---
title: RocketMQ消息队列(一)架构
tags:
  - 消息队列
  - rocketmq
  - 消息队列
  - kafka
description: RocketMQ是阿里巴巴开源的一个消息队列，本系列文章会对RocketMQ的技术架构、功能等做深入的研究。
categories: 消息队列
toc: true      
date: 2020-04-28 18:07:34
---

:warning: 原创文章，转载请注明出处

## [](#架构概览 "架构概览")架构概览

RockeMQ 主要包含两个模块 Broker和Name Server，如下图所示:

![](/2020/04/28/RocketMQ消息队列-一-架构/rocketmq_arch.svg "rocketmq arch")

### [](#NameServer "NameServer")NameServer

NameServer是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。

主要包括两个功能：

1.  Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；
    
2.  路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。
    

然后Producer和Consumer通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。

NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。

当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。

### [](#Broker "Broker")Broker

Broker主要负责消息的存储、投递和查询以及服务高可用保证。broker会每隔30s向集群中的所有nameserver发送一个心跳包，nameserver会每隔10s扫描自己保存的broker列表，看broker最后一次发送的心跳包是否是120s前的，如果是就删除这个broker，关闭链接。

### [](#生产端和生产端集群 "生产端和生产端集群")生产端和生产端集群

Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic 服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。

group name相同的一组生产端，称之为生产端集群。集群内每个生产者都会给master发送心跳，所以master是掌握所有生产者信息的，在事务消息回查时，broker端可选择生产端集群中的一个，来执行回查逻辑。

### [](#消费端和消费端集群 "消费端和消费端集群")消费端和消费端集群

Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，消费者在向Master拉取消息时，Master服务器会根据拉取偏移量与最大偏移量的距离（判断是否读老消息，产生读I/O），以及从服务器是否可读等因素建议下一次是从Master还是Slave拉取。

groupname相同的消费端，称之为一个集群。集群内每个消费者都会给broker发送心跳，所以broker端也掌握了所有消费者的信息，每个消费者上线、或下线时都会来查阅这个信息，进行队列重分配。

### [](#Broker集群 "Broker集群")Broker集群

RocketMQ的集群比较特殊，是多个单元组成的一个集群。如上图所示，整个集群包含5台broker，两个单元，第一个单元是3台broker，一主两从，第二个单元是一主一从。集群的划分是以cluster name名称为准备，命名相同的机器都属于一个集群。如上图，所有broker的cluster name属性都叫order-cluster，他们都属于一个集群。

name相同的一组broker是一个单元，同一单元内，id属性为0的broker是master，id属性为1的为第一slave，其他都是slave.

## [](#Topic和队列概念 "Topic和队列概念")Topic和队列概念

Kafka里每个topic各自的partition消息，都会写入自己的文件里。RocketMQ不一样，它把所有的topic数据全部写入一个文件里，称之为commit log。但消费的时候怎么区分每个队列呢？答案是broker接收到消息后，统一都写入一个消息日志(commit log)文件，由转发服务(reput Service)再转发生成消费队列(consume quue)文件，如下图所示:

![](/2020/04/28/RocketMQ消息队列-一-架构/rocketmq_queue.svg "rocketmq concept")

上图可以看到有两个文件：

(1) CommitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

(2) ConsumeQueue：消息消费队列，引入的目的主要是提高消息消费的性能，类似于kafka中的partition概念，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。Consumer即可根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。consumequeue文件可以看成是基于topic的commitlog索引文件。同样consumequeue文件采取定长设计，每一个条目共20个字节，分别为8字节的commitlog物理偏移量、4字节的消息长度、8字节tag hashcode，单个文件由30W个条目组成，可以像数组一样随机访问每一个条目，每个ConsumeQueue文件大小约5.72M；

还有一个文件是:

IndexFile：IndexFile（索引文件）提供了一种可以通过key或时间区间来查询消息的方法。Index文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。

### [](#队列分布方式 "队列分布方式")队列分布方式

了解了RocketMQ中 topic和consume queue概念，以及rocketmq集群搭建方式后，就很容易了解 rocketmq队列的分布方式了，以图1中2 master 3 slave的5 台机器组成的集群为例，比如创建一个topic 和6个队列(consume queue)，可以如下来分布。

![](/2020/04/28/RocketMQ消息队列-一-架构/rocketmq_queue_deploy.svg "rocketmq deploy")

brokerA的master节点有3个队列， brokerB的master节点有3个队列，这里的队列类似Kafka中的partition概念。