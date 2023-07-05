layout: _post
title: Apache pulsar对zookeeper的依赖和去除分析(一)
date: 2023-06-15 17:13:00
description: Apache pulsar对zookeeper的依赖和去除分析(一)
tag: Apache pulsar 消息队列 kafka RocketMQ
toc: true
---

# 背景
Apache pulsar是近年来国内比较时髦的消息队列，而且是开源的产品，有不少国内的互联网公司都开始了使用了。针对开源的几个消息队列产品比较，可以参看我之前的一篇博客[<<消息队列业界调研>>](/2021/03/26/消息队列业界调研/ "消息队列业界调研")。Pulsar为什么现在比较火，说白了是搭上了"云原生"这趟车。在当今时代，任何互联网技术，特别是基础组件，如果没云原生化，出门别人都不好意思给你打招呼。特别是从2021年开始， 突然一切互联网技术都需要"云原生"了，其中隐情也是各有使然，当然这是另一个话题。(关于我在美团时负责的消息队列Mafka云原生化分析，可以参考我的这篇博客和 [<<消息队列Mafka列长期发展计划>>](/2021/07/14/Mafka-LRP/ "Mafka消息队列长期发展计划") 和 [<<消息队列Mafka全链路弹性伸缩演进策略>>](/2021/07/22/Mafka全链路弹性伸缩演进策略/ "Mafka全链路弹性伸缩演进策略")  )Pulsar因为其"天然"的存算分离架构，和云原生讲究的弹性伸缩性(scalability)，品性特别符合，自然受到了大家特别的追捧。

关于pulsar架构的评价，可以参考我之前写的这篇博客[<<消息队列业界调研>>](/2021/03/26/消息队列业界调研/ "消息队列业界调研")。本篇文章主要分析下pulsar对zookeeper的依赖，因为在当前的架构下(本篇以pulsar 2.8版本分析)，pulsar对zookeeper是强依赖关系，说白了就是zookeeper一旦挂掉，pulsar整个集群也就挂掉了。因为之前我也有一篇博客分析kafka 去除zookeeper依赖[<<揭秘Kafka去Zookeeper技术>>](/2021/12/01/Kafka去Zookeeper揭秘/ "Kafka去Zookeeper揭秘")，本篇分析下pulsar对zookeeper的依赖，以及如何去除。因为涉及的内容比较多，会分为三篇博客来写，第一篇分析pulsar本身，即pulsar broker，第二篇分析 bookkeeper对zookeeper的依赖，第三篇 是临时加的，因为2023年5月份，pulsar官方出了一个类似kafka kraft的一致性组件([moving-toward-zookeeper-less-apache-pulsar][1])来去除pulsar对zookeeper的依赖，第三篇会重点分析下这个组件。

[1]: https://streamnative.io/blog/moving-toward-zookeeper-less-apache-pulsar "moving-toward-zookeeper-less-apache-pulsar"

# 架构简介

## 概览

在讨论细节之前先来重温下pulsar的架构，pulsar架构的详细介绍，可以参考我之前的博客[<<消息队列业界调研>>](/2021/03/26/消息队列业界调研/ "消息队列业界调研")一文
![](pic5.svg)
1. 整个pulsar 分为broker和bookkeeper两大部分
2. broker 和 bookkeeper都依赖 zookeeper来做数据存储
3. broker是无状态的计算层(有部分配置缓存和消息数据缓存)
4. bookkeeper是存储层, bookkeeper实际上是apache的另一个开源项目

## 几个实体
1. topic
    topic就是一个队列，pulsar中的 topic 对应bookkeeper中的一个ledger。多partition的队列就是多个队列组成的，每个topic对应一个partition
2. group
     消费组
3. bundle
     队列集合。pulsar 使用hash算法，将多个队列归为一个集合，叫做bundle。bundle是一个虚拟实体，bundle 会被分配到不同的集群节点上，每个bundle 包含一部分队列。要计算topic在哪台节点上，需要先计算topic在哪个bundle里。 pulsar管理集群节点和topic的归属关系时，是以bundle为粒度的。
4. tenant
    租户
5. namespace
    ns空间
6. managed leger
    pulsar包装的bookkeeper中的ledger
7. managed cursor
     pulsar抽象化的消息队列指针
8. policy
     namespace空间内使用的集群规则项目
9. local policy
     本机群内所有bundle的信息等


# Broker对Zookeeper的依赖总结
先来看下broker对zk依赖的总结列表
| 项目                                            | zk 路径                                               | value/children  | 备注                                                                                         |   | 
|-------------------------------------------------|-------------------------------------------------------|-----------------|----------------------------------------------------------------------------------------------|---|
| pulsar polices定义都集中在/admin/polices目录下 | /admin/policies                                       | children        | 租户列表                                                                                     |   |
|                                                 | /admin/policies/[tenant]                              | children        | namespace列表   初始化创建两个namespace： 1. public/default/ 2. pulsar/system/存放system topic |   |
|                                                 | /admin/policies/tenant/[namespace]                        | value           | namespace 级别policies策略                                                                   |   |
| bundle相关的数据                                | /admin/local-policies/[namespace名称]                 | value           | 集群内所有的 namespace下的所有bundle信息   节点的value值实际上是 LocalPolicies类的序列化     |   |
|                                                 | /namespace/tenant/[namespace]                              | 临时节点  value | bundle的owner信息 NamespaceEphemeralData[java类]: 节点服务信息ip和port一类                   |   |
| clusters                                        | /admin/clusters                                       | children        | cluster列表                                                                                  |   |
|                                                 | /admin/clusters/[cluster]                             | value           | cluster 信息数据 clusterData                                                                 |   |
|                                                 | /admin/clusters/[cluster]/ namespaceIsolationPolicies | value           | namespace isolation polices集群分组策略                                                      |   |
|                                                 | /admin/configuration                                  | value           | 动态配置 DynamicConfiguration                                                                |   |
| brokers                                         | /loadbalance/brokers                                  | children        | active brokers在线的broker                                                                   |   |
|                                                 | /loadbalance/brokers/[ip:port]                        | 临时节点 value  | LocalBrokerData[java类]                                                                      |   |
| topics                                          | /managed-ledgers/[tenant]/[namespace]/persistent      | children        | topic信息                                                                                    |   |
|                                                 | /admin/partitionedTopic/[topic全名]                   | value           | topic的partition信息                                                                         |   |
| topic的ledger、cursor、消费组相关信息           | /managed-ledgers/[topic全名]                          | value           | topic的 ledger 数据，实际是java类 ManagedLedgerInfo的序列化                                  |   |
|                                                 | /managed-ledgers/[topic全名]                          | children        | topic 的消费组名称(同时也是cursor名称/cursor ledger名称)列表                                 |   |
|                                                 | /managed-ledgers/topic全名/[消费组名称]               | value           | 消费组的cursor leger 信息 实际上是java类 ManagedCursorInfo的byte形式                         |   |

粗略分的话，大约分以下几类：
1. topic相关信息： topic/group/bundle/ledger/cursor一类，这一类信息都是和topic相关的，但是集群在操作的时候都是以bundle粒度来操作的。要确定topic在哪台机器上，先确定topic属于哪个bundle.
2. broker相关信息: 集群内有多少个节点，每个节点当前的状态
3. 配置类信息：tenant/namespace/cluster/动态配置/policy配置，这些信息都是涉及到整个集群配置的，创建、删除、修改都依赖zk的增删改查

# 分析
## 1.topic相关信息
上边说过，pulsar集群在操作topic时，是以bundle为粒度，一个bundle包含一批topic。bundle 被集群内的不同节点持有，哪个节点持有某个bundle，它就负责这些topic相关的操作，比如创建和删除topic /group/ledger/cursor等信息。

bundle 信息存储在zk上两个节点下:
- `/admin/local-policies/[namespace名称]` 下Value值，存放着localPolicies对象的序列化信息，localPolicies对象里含有bundleData信息，bundleData主要包含 namespace下所有bundle的数量，每个bundle之间的界限范围列表，比如(0x00000000，0x100000000，0x200000000)

- `/namespace/[namespace名称]/[localhost:8080]/[0x00000000_0xffffffff]` ，即/namespace/[bundle名称]的Value 值，是一个临时节点，存放了SelfOwnerInfo(java类，包含nativeUrl,httpUrl)，就是包含这个bundle的节点信息

同时这两个节点也会在`NamespaceBundleFactory`和`OwnershipCache`中形成一个cache信息，跟bundle关心紧密的类还包括`NamespaceService`
这个类主要是来服务bundle信息的，比如确定bundle在哪台broker上，获取一个bundle等，确定topic属于哪个bundle。

### 多个节点并发写zk导致的一致性挑战: bundle信息并发写 和 bundle所有权多节点并发抢占
因为 bundle 在 pulsar内被设计为可以在多个节点之间漂移的，同时也是可以被分裂成多个bundle的，所以bundle信息在zk上的维护会在集群内多个节点之间产生并发操作

#### 当前节点加载被访问到的 namespace，和namespace下拥有的所有bundle:
```
/admin/local-polices/[ns1]     -> [value]  localPolices
                    [ns2].
                    [ns3]·
```
namespaceBundleFactory:    
自己的重要属性bundlesCache，这个cache内有每个namespace和其所有的bundle组成的KV缓存。
这个cache 也是异步加载的，需要人为来触发，当来一个lookup请求，需要知道哪个机器负责这个bundle时，就会触发当前topic所属namespace的缓存加载，加载时会load当前namespace所包含的所有bundle信息。这个KV缓存不止缓存本机拥有的bundle的namespace，他会包含所有被服务过的toipc的namespace。

缓存加载doLoadBundle:  从zk上读取bundle信息，反序列化成localPolicies，同时会保留一个pulsar自己的通过zk state构造的Sate里的version信息，是long型值，这些信息都会保存在KV缓存Value 里的Namespacebundles对象内， 将来做bundle拆分时，需要将拆分后的bundle信息回写到zk上时，这时会产生多个broker节点并发写zk的情况，因为这里存储的是某个namespace负责的所有bundle信息，其他broker也有拿到这个namespace下某一个bundle的所有权，做bundle拆分时，也会更新bundles 总信息，这时就要有一个一致性协调者zk，更新zk信息时，需要比较上次缓存的version信息和zk上现在的version信息时是否相等，避免覆盖其他人写的值。这个version在这里的作用，其实还是为了确保自己缓存的zk 数据信息是最新，防止将旧的数据会写到zk上。

#### 加载当前节点自己拥有的bundle列表
```
/namespace/
    [ns1]/[0x00000000-0xfffffff0]    -> [value] OwnedBundle
    [ns1]/[0xfffffff0-0xffffffff]
```

NameSpaceService里的重要属性ownerShipCache对象，这个对象拥有一个重要的KV缓存属性，key是NamespaceBundlezNode，value是ownedBundle，简单说key就是一个bundle在 zk 上的路径(/namespace/[namespaceName]/[bundleRange])，zk上对应的value是bundle对象的序列化。这个缓存保存了当前节点拥有的bundle列表，这些信息在OwnerShipCache里保存了一份，同时也会保存在zookeeper上。当前broker 负责的bundle信息，在zk上保存的是一个临时节点，因为当前broker如果宕机了，zk链接就会断开，临时节点就会丢掉，当前broker释放bundle的所有权，以期望其他broker能获取。

注意这个cache不是自动创建的，必须有人来触发他，比如有一个lookup请求打到这个节点上时，broker 需要直到这个topic所属于哪个bundle，并且会从本机的这个缓存里查找一下，看自己是否拥有这个bundle。如果拥有，则正常返回。如果不拥有这个bundle，则会再从zk上查一下谁当前拥有这个bundle。这里查询的时候，如果查到这个节点已经存在，则会比较下节点创建的zk sessionId，和当前的sessionId是否相等，并且负责这个bundle的broker的url是否也相等，如果都相等的话，则认为是自己负责的。注意，这时其实发生了zk和本机缓存不一致的情况，zk上的临时节点是自己创建的，但是本机缓存却不存在，这里实际上是做了一次补偿，来统一zk数据和本机缓存数据。如果不想等，则会认为是其他人拥有这个bundle，会将请求打到真正负责这个bundle的broker上。如果没人拥有这个bundle，会触发让leader选取一个合适的候选节点来负责这个bundle。如果候选节点是本机，则当前broker会尝试获取这个bundle，把这个bundle负责起来。
尝试获取bundle的时候，实际发生了 zk 的并发抢占，当其他节点也发现无人负责这个bundle时，很有可能也去抢占获取这个bundle，所以这时需要一个zk的一致性机制来保障。

copy polices to local polices: 把polices里的bundle信息保存到local policies里，保存时，期望zk里的version是 -1，其目的是表明是自己来完成初次的local polices的构造，防止覆盖了其他人的写。

**总体来看**，<ins>namespace下所有bundle信息的维护，bundle的拆分，以及拆分后每个bundle的所有者分配，完全可以交由集群的leader来维护，避免多个节点都去执行写操作，产生并发写竞争。避免因为并发写，而必须引入zookeeper这个一致性协调者。所有的写操作，都可以交由leader来完成，leader和其他节点的通信，使用epoch方式来保障，确保集群内所有的节点对当前leader 有统一和一致的认识，这点可以参考Kafka类似的系统。</ins> 

**当bundle信息确定后**，<ins>每个节点都有负责的一组固定的bundle，一个bundle对应着一批topic，topic的ledger信息、cursor信息、group 信息都有topic所在的节点自己来负责，对zk的读写操作基本都是低频的，即便发生并发，在单节点、单jvm内使用普通的java并发工具包类也是很好控制的，基本不涉及到多个节点的并发。因此这类操作，基本是拿zk当存储来使用，并没有使用到zk的全局一致性保障功能。</ins>

### 附 topic相关的操作:
<div style="color: #6E7173">

ledger、cursor 相关的topic操作列表，以及访问zk的操作
1. topic创建      
   创建ledger，保存ledger信息到zk
2. group创建
   创建cursor ledger，保存cursor ledger信息到zk。分区topic 的 group创建是分别给每个分区的topic轮流创建一遍。
3. producer创建
   无zk操作。分区topic的producer创建也是按分区个数轮流创建。
4. send操作
   当前ledger添加满之后，创建新的ledger，保存新的ledger信息到zk
5. consume操作
   消息拉取委托managedCursor来完成，本身无相关的 zk 操作。
6. 客户端ack消息
   ack的消息和位点信息，最终会构造为一个PositionInfo的对象，序列化之后添加到cursor ledger中，cursor ledger虽然也是一个ledger，但不同的是这个ledger只会有一个，当当前ledger添加满了之后，topic的ledger会负责创建一个新的ledger来存储位点信息。 

这些都没有涉及到zk的操作。只有创建cursor ledger时，需要将这个ledger信息保存到zk上。
同时，当ack消息添加到cursor ledger失败时，或关闭cursor时，或cursor ledger发生切换时，才会将位点信息补写一份到zk的cursor 节点value中。

</div>

## 2.broker 节点相关的操作
broker 节点相关的zk操作基本都在`/loadbalance/`节点下，节点上线、下线会导致创建或删除临时节点，这一部分和负载均衡的逻辑在一起。
所有 broker会定时将本机的负载数据，写到zk上，路径是`/loadbalance/broker-time-average/[ip]`，内容是LocalBrokerData类对象的json序列化。
集群会首先选出一个leader，leader会启动一个loadSheddingTask和一个LoadResourceQuotaTask，前者会根据集群当前的负载信息卸载需要均衡的bundle，后者会综合所有 broker 上报的系统负载信息和bundle信息计算出长期、短期两个维度的负载信息，再更新到zk上，zk 路径是`/loadbalance/broker-time-average/[ip]`，内容是TimeAverageBrokerData对象的json序列化。这里leader broker和普通的broker往 zk 上上报的信息，使用的是不同节点，不涉及到zk的并发写，因此这里仍然是拿zk作为一个存储来使用的。

## 3.配置相关的操作
tenant/namespace/cluster/动态配置/policy配置，这些信息都是涉及到整个集群配置的，创建、删除、修改都依赖zk的增删改查。这些信息都是集群的静态配置，目前是每个集群节点都可以去操作zk来处理这些请求的，
同样也是通过zk的返回代码来判断是否有并发写，依据前面topic信息一类的分析，仍然是可以由集群leader来独自完成的，避免对zk的并发写。

# 思考
如果使用epoch的方法，所有的集群操作都由leader来完成，leader 会不会变成一个单点风险？ 确实会，不过这时可以用zk来做一个抢占临时节点来选出集群leader，这是唯一需要zk的地方，其他所有的集群写操作都由leader来完成，需要存储的数据由关系数据库或kv来完成(这一项我们在美团递消息队列Mafka castle的模块改动上有实践过)。

如果都由leader来完成，leader会不会称为一个瓶颈点？不会，总结上边会触发zk写操作的事件，相比收发消息一类的请求来说，都不是一个量级的，所以leader节点不会是一个瓶颈点。如果实在担心，可以将leader节点设置为纯 leader，不承载任何 bundle 数据，不接受消息收发类的数据请求，这点我们在美团消息队列Mafka上使用过。还可以将数据请求操作和集群管理操作做分类，leader 节点在处理请求时，优先处理管理类操作，也可以减轻leader节点的负担，就像是Kafka在1.1.0版本后的改动一样。






