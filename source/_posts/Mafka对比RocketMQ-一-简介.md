---
title: Mafka对比RocketMQ(一)简介
tags:
  - 消息队列
  - rocketmq
  - 消息队列
  - kafka
description: Mafka是美团基于kafka自研的消息中间件，RocketMQ是阿里巴巴开源的一个消息队列，本系列文章会对Mafka和RocketMQ的技术架构、功能等做深入的对比和研究。      
categories: 消息队列      
date: 2020-04-28 18:10:16
toc: true
---

:warning: 原创文章，转载请注明出处
## [](#Mafka简介 "Mafka简介")Mafka简介

### [](#什么是Mafka "什么是Mafka?")什么是Mafka?

Mafka是基础架构-MQ团队从2016年开始自研的消息队列产品，底层基于Apache Kafka，增加了自研的基于机房粒度的中心化调度、时间回溯、粘性分配、死信、延迟队列、适用于美团自用的同步/异步客户端、机房容灾等高阶特性，目前版本是3.0。

### [](#为什么建造Mafka "为什么建造Mafka?")为什么建造Mafka?

消息队列是现代互联网技术架构中不可缺少的一个基础组件，对于美团这样的业务多样性的大型互联网公司来说，这个组件尤其变得更为重要。2016年Mafka诞生之前，美团拥有自研的消息队列产品Nuclear MQ和Swallow，但是随着公司业务的不断扩大，业务线对消息队列功能多样性，以及可用性，可靠性，高并发能力的要求越来越高，原有队列产品的局限性和先天不足劣势迅速暴露出来。正是在这样紧迫形势下，MQ团队经过深入调研和分析，发现原有产品的基础技术架构已无法满足用户需求，决定基于开源社区比较成熟的消息队列产品Apache Kafka研发打造适用于美团的消息队列产品，Mafka就这样诞生了。

Apache Kafka诞生于同样是互联网公司的LinkedIn，经过企业级互联网产品的实战检验，拥有优秀的技术架构和设计。但Kafka的定位是作为Apache基金会下一个开源的消息队列产品，其功能和设计演进完全是集所有公司和用户需求的基础共性为重点的，涉及到具体某个公司的个性化或特定环境下的需求和演进，并不是他演进的方向。所以直接拿Kafka来部署，作为美团的企业级消息队列产品是没法使用的。所以，MQ团队对Kafka做了深度的定制和自研，来满足美团的业务场景。为了区分开源产品Apache Kafka和经过MQ团队改造后的Kafka，MQ团队将后者命名为Mafka。

Mafka 基于Kafka 0.9.0.1版本研发，至今已迭代了3个版本(第四个版本正在迭代中)。开源Kafka只有Broker和clientSDK两个模块，实际真正使用到美团这样的多样性大型互联网O2O公司内，其原生设计已经远远满足不了业务的功能需求以及企业级产品标准。Mafka除了对Kafka原有的Broker和java clientSDK两个模块做了深度自研和定制外，还增加了Castle、延时队列、Push消费组、Mafka Scanner、Auth队列鉴权、全模块自动测试系统、c++/php客户端、用户平台、运营平台等9个模块的系统支持。其中，仅仅是Broker模块，新增代码将近4万行，删除代码近9000行，涉及800多次提交。整个Mafka 系统新增代码量数十万行。

### [](#Mafka架构图 "Mafka架构图")Mafka架构图

![](/2020/04/28/Mafka%E5%AF%B9%E6%AF%94RocketMQ-%E4%B8%80-%E7%AE%80%E4%BB%8B/mafka_arch.svg "Mafka架构图")

## [](#RocketMQ简介 "RocketMQ简介")RocketMQ简介

### [](#rocket演进历史 "rocket演进历史")rocket演进历史

![](/2020/04/28/Mafka%E5%AF%B9%E6%AF%94RocketMQ-%E4%B8%80-%E7%AE%80%E4%BB%8B/rocketmq_iter.png "rocketmq演进历史")

> 2007年，淘宝实施了“五彩石”项目，将交易系统由单机交易升级到了分布式，这个过程中产生了Notify。  
> 2010年，阿里巴巴B2B部门基于ActiveMQ的5.1版本也开发了自己的一款消息引擎，称为Napoli。  
> 2011年，Linkin推出Kafka消息引擎，阿里巴巴在研究了Kafka的整体机制和架构设计之后，基于Kafka的设计使用Java进行了完全重写并推出了MetaQ 1.0版本，主要是用于解决顺序消息和海量堆积的问题，由开源社区killme2008维护。  
> 2012年，阿里巴巴对于MetaQ进行了架构重组升级，开发出了MetaQ 2.0，这时就发现MetaQ原本基于Kafka的架构在阿里巴巴如此庞大的体系下很难进行水平扩展，所以在2012年的时候就开发了RocketMQ 3.0。  
> 2015年，又基于RocketMQ开发了阿里云上的Aliware MQ和Notify 3.0。  
> 2016年，阿里巴巴将RocketMQ的内核引擎捐赠给了Apache基金会。

MetaQ和RocketMQ区别：两者等价，在阿里内部称为MetaQ 3.0，对外称为RocketMQ 3.0。

以上就是RocketMQ的整体发展历史，其实在阿里巴巴内部围绕着RocketMQ内核打造了三款产品，分别是MetaQ、Notify和Aliware MQ。这三者分别采用了不同的模型：

> MetaQ主要使用了拉模型，解决了顺序消息和海量堆积问题。  
> Notify主要使用了推模型，解决了事务消息  
> 而云产品Aliware MQ则是提供了商业化的版本。

从阿里内部同学获取到的信息，阿里内部使用多套消息队列服务，并不统一，有三套消息队列服务: notify, metaQ(kafka),RocketMQ。没有特殊需求时，一般使用metaQ，需要事务消息时，使用notify，需要定时消息时使用RocketMQ.

RocketMQ架构图

![](/2020/04/28/Mafka%E5%AF%B9%E6%AF%94RocketMQ-%E4%B8%80-%E7%AE%80%E4%BB%8B/rocketmq_arch.svg "rocketMQ 架构")