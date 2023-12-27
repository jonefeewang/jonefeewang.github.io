layout: _post
title: Apache Pulsar的存储模型
date: 2023-12-25 17:13:00
description: 本文分析了Apache pulsar的底层存储模型
tags:
  - Apache Pulsar
  - Apache Bookkeeper
  - Apache Kafka
categories: 消息队列
toc: true
---

# 引言
Pulsar实际上是三个组件的组合体Apache Pulsar+Apache Bookkeeper+RocksDB。 Bookkeeper是负责消息的存储，Pulsar负责队列和消费的概念模型塑造，RocksDB是Bookkeeper内部用来存储kv索引用的，相关的概念介绍，可以参考下我之前的博客:[<<消息队列业界调研>>](/2021/03/26/消息队列业界调研/ "消息队列业界调研")。本篇来详细分析一下Pulsar的消息存储模型。


# 概述
其实说到底，Pulsar的消息存储最底层就是ledger，这个概念也是Bookkeeper中的概念，ledger由很多小段的segement组成，这些segment在bookkeeper中称之为entry log，所以要彻底说清楚Pulsar的存储，还得由Bookkeeper来说起。entry log是什么？是不同的entry写到一个文件里，这里的entry可以理解为消息，一个entry从属于某一个ledger。所以，反过来说，一个ledger由entry组成，不同的ledger在存储时都写入了一个称为entry log的文件里。

Bookkeeper本身是一个有限的流式数据存储，这里说的有限是对应与Kafka的无限流式数据存储。kafka中一个log流是有开始没有结束，一个log代表一个topic的一个partition，只要队列有消息，这个流就不会终结，永远存储在这个log文件里(具体文件会滚动，但存储的都是这个partition的消息)。而bookkeeper则不同，它是一个有限的流式存储，一个ledger代表队列的一个partion数据，他有开始也有结束，开始就是有消息生产就有了开始，结束时是当消息的生产者掉线了，死机了，或是生产这主动关闭了，这时就是这个ledger的终点，这时候ledger会被关闭掉，如果下一个生产者继续往这个partition里生产，则需要创建一个新的ledger。在整个ledger的生命周期内，它只有三种状态，open/in_recovery/(fenced)closed，

* open：在正常接收生产的消息数据
* in_recovery: 读取客户端在试图恢复之前未被复制到所有WQ的消息，这时停止接受新的消息数据。
* (fenced)/closed: 恢复完数据后，客户端关闭这个ledger，不再接受任何新的消息数据。

bookkeeper对生产端有两个严格的要求:
1. 第一个是任何时间段必须只能由一个客户端在生产，这也就是为什么pulsar里bundle只能被一个broker负责，不能由不同的 broker同时负责，那样就会有两个生产端同时生产了，会产生脑裂，违反了bookkeeper的要求。
2. 第二个是生产时必须提供一个顺序的entryId，也就是消息Id，这个限制是方便对ledger内的fragment的进行按顺序的消息管理，比如0-100的fragemnt的ensemble是(B1,B2,B3)，100-500的ensemble是(B2,B3,B4)，如果entryId不是顺序产生的，这个fragment是无法归类的，到底哪一段，哪一个范围的entry是哪些ensemble负责的。确定ensemble的信息非常重要，因为消息在读取的时候，需要这个信息，比如要读取entryId是233的消息，有了这个fragment信息，我就知道从哪个bookkeeper机器上读取了。

再说回到pulsar，因为bookkeeper只提供了ledger这个概念，根本没有消息队列里的topic/group/cursor这些概念，直接拿到pulsar里使用是无法使用的，所以pulsar在ledger的基础之上封装了一次，将topic的partiton封装为一个ledger，在pulsar内部叫做managedLedger，一个partition包含多个ledger，因为上边讲到ledger是一个有限数据流，因为broker死机，或者entry log的size达到上限，都需要结束这个ledger，重新create一个。所以，在puslar里，一个多partiton的topic，每个partition其实是多个ledger组成的，这些信息存储在zk的目录上，具体参考我之前的博客[<<Apache pulsar对zookeeper的依赖和去除分析(一)>>](/2023/06/15/Apache-pulsar对zookeeper的依赖和去除分析-一/ "Apache pulsar对zookeeper的依赖和去除分析(一)")。

所以，pulsar自身并没有存储消息数据，所有的消息数据，都被存放到了bookkeeper上，所有抽象出来的topic/group/cursor元信息内容都被存储到了zk上，它自己本身是无状态的。

# Bookkeeper存储的设计和实现

bookkeeper生成的物理文件有三种，一个是journal，一个就是entry log，一个是logmark文件
* journal：类似zookeeper中的transaction log，或者是write log，短期存储，，会按照大小分割和滚动,所有ledger发送的消息都会在这里实际落盘后，才会ack客户端成功。
* entry log: 实际存储entry的文件，是一个长期存储，所有的ledger都会写到一个entry文件内部，相同ledger的entry会做排序，方便读取的时候顺序读取相同ledger的entry。(实际内部还有另外一个实现option，每个ledger一个entry log，不过因为ledger很多，这样会产生很多碎文件)
* logmark文件：标注journal文件内一个位置点，比如是哪个journal文件内的哪个offset点。

每个ledger还有一个masterKey，它是一个ledger的访问口令，在ledger生成时会指定一个，以后ledger的访问都会在请求中校验这个key。

## entry的写入逻辑
Bookkeeper使用Netty，netty worker thead解析到是addEntry(添加一条消息)请求后，交由write thread pool来处理具体的entry插入。write thread pool是一个ordered Excutor线程池，所谓ordered Executor，意思所有属于同一个ledger的entry 插入都会交由同一个固定的线程去负责执行，以减少乱序和并发。每个ordered thread的任务池大小，以及write thread pool大小都可以在配置文件中设置。由此可见bookkeeper在插入entry时，并没有批量插入的接口，orderedExcutor的固定线程在执行queue中的任务时，虽然是批量获取，执行时仍然是一个一个的执行插入，并没有类似kafka中的批量消息插入。

ordered executor在执行具体某个ledger的entry插入任务时，分两步进行，第一步是插入EntryLog，第二步是插入Jornal，在jornal插入并刷盘后，再回调entry的插入成功逻辑，ack客户端插入成功。

插入entryLog时，实际上第一步先插入到WriteCache内，这是一个分段cache，有两个，一个是当前正在被使用的(active)，另一个是当前正在被flush到磁盘上(in_flush)。active的就是平常entry先插入到这里，in_flush的是正在flush WriteCache内的entry到磁盘上，flush完成后这个cache会被清空，等待active的cache插满之后，和它做个交换，以保障正常的插入流程能继续进行，当然如果flush的慢的话，也会形成反压，导致WriteCache的插入被block，进而导致插入请求超时。同时， active 的WriteCache插满之后，会做checkpoint，所谓的checkpoint就是一个时间点，这个时间点会在Journal内做个标记，标记一下Journal最后一次flush 磁盘时插入的entry所在的journal文件id和position，称为LogMark,就是上边讲的log mark文件存储的内容，在WriteCache内的数据被flush到磁盘后，entry log会通知jounal，当前时间点之前的entry已经完全flush到磁盘上了，不需要journal内的数据了，可以将刚才做的checkpoint之前的记录删除掉，腾出磁盘空间。

由此可见，entry的插入，在Jornal之内是同步插入的，而entrylog则是异步插入的, Jornal保障数据持久化之后，会ack客户端插入成功。entrylog虽然是异步插入，但是可以通过write cache类形成对客户端的反压，block 所有entry的插入请求。Journal有了checkpoint之后，在单台bookkeeper宕机时，因为entry log是异步flush磁盘的，有可能部分entry还未flush而丢失，这时，就可以根据checkpoint从journal内做一次replay，把丢失的entry给找回来。

entry在插入entry log时，会保留位置信息，也就是所在entry log文件的id，和position，位置信息的key是ledger id+ entry id, value是 entry log文件的id和文件内position组合起来的一个lang值，将这个key value pair保存到rocks db内(或index文件内，见下文)。

### WriteCache的实现原理

bookkeeper的配置文件内，会设置最大Write Cache大小，每段Cache的段大小，因为在内部，整个缓存区域是被分成段来缓存的，bookkeeper默认设置每段缓存的大小是1G，所以如果总缓存大小是2G的话，每段大小是1个G。
使用分段来缓存具有很多优势:
1. 提高读写速度，因为每段的缓存更小，可以更快的定位entry所在的位置，进行快速读取
2. 方便并发控制，这个原理跟java语言中的ConcurrentHashMap类似，不再赘述
3. 还可以减少内存碎片，因为每个小段可以作为整体来分配和回收，整体利用，减少了碎片的形成。

在写入Cache时，会先计算一个全局offset，设置size大小为align64，就是接近64的最小倍数，防止在CPU L1 Cache产生线程竞争。为什么是64？因为java中Long型的长度是64位，使用Long型来计算Bits码，防止Cache最小段大小超过Long型所能表示的最大值。然后使用Bits码和掩码，快速定位到应该放到哪一段，和段内的offset(这个算法是存储系统经常使用的方法)。

Cache 内部保存了一个HashMap，是一个(ledgerId,entryId)到(offset,size)的映射，注意这里的offset是全局offset，size是这个entry的大小字节数量。

读取cache内的某个entry的时候，使用全局offset反算出entry所在的段和位置。

### entry log文件
默认的entry logger 是使用java的文件I/O来写entry log文件，bookkeeper额外提供了使用JNI来写文件的DirectEntryLogger实现，来加快文件写的速度。使用java文件I/O实现的entry logger还有两种方式，一种是将entry log全部写进一个文件中，另外一种是每个ledger一个entry log，前者是默认的。

entry log文件存储的管理，由bookkeeper内部抽象出来的LedgerStorage来管理，这个接口有3种实现:

1. InterleavedLedgerStorage 没有使用内存缓存最近插入的entry，entry在写入和读取直接进行文件操作。索引存储使用了多页的jvm直接内存来缓存，缓存的数据是ledger id+ entry id -> entry log id 和 entry log内 position组合成的long型数据

2. SortedLedgerStorage  是InterleavedLedgerStorage的扩展，在它的基础上加了一层最近插入的entry内存缓存，缓存内部使用跳表map来实现，一共两个，一个作为当前在插入的缓存，另外一个作为后台flush entry到文件时使用，flush完成后，缓存清空，等待active的写满之后与它做交换。

3. DbLedgerStrage 使用WriteCache和ReadCache来缓存最近写入的entry和最近读取的entry，索引存储使用rocksDB来做kv存储，所以取名DBLedgerStorage，也是上面两节讲到的entry插入流程中使用到的。每次WriteCache写满之后，触发Journal checkpoint，然后触发ledgerStorage checkpoint， ledgerStoarge checkpoint时会flush数据到磁盘，然后通知checkpoint source自身已完成，这时checkpoint source是Jornal，journal收到通知后，删除旧的journal日志。(DbLedger Storage的set checkpointer()方法是空的，虽然传进来的SyncThread但是没设置，所以在dbledgerStorage内，syncThread并未起到作用。)

### journla的写入
entry交给journal后会暂存在journal的缓冲队列中，交由 journal的专用thread来驱动写入文件，到达一定条件后flush文件到到磁盘上，最后再ack entry的插入操作算成功。jornal触发批量刷盘的条件:
1. group commit等待的时间超过了设定的时间 
2. 缓存的entry数量超过了设定的值
3. 缓存queue内元素暂时为空
当这些条件中的某一个达到时，一般会生成一个forcewrite request，放到force write thread的任务队列里，这个thread会一直运行，只要队列内有请求就会执行flush操作。执行的第一步先flush journal文件到此盘，然后挨个回调被flush 的entry的回调函数，在回调函数内，ack给客户端entry插入成功。

syncthread
这是一条独立运行的thread，他会定期运行，先对ledger storage进行checkpoint，其实就是flush active write cache内的entry到磁盘，，然后通知jornal的,让journal删除checkpoint之前的文件，清理空间。


## entry的读取
Netty worker thread在解析到是read entry操作时，会交由专门的read thread pool处理读取请求。如果是long poll read会交由专门的long poll thread pool来处理，如果请求的header内包含priority >0，则表示是一条high priority read请求，会交由专门的high priority pool来完成，以保证高优先级任务能顺利完成。
entry读取的请求实际上包含了多种请求:
```
ReadRequest{
    enum Flag{
        FENCE_LEDGER=1;
        ENTRY_PIGGYBACK=2;
    }
    Optional Flag flag=100;
    required int64 ledgerId=1;
    required int64 entryId=2;
    optional bytes masterKey=3;
    optional int64 previousLAC=4;
    optional int64 timeOut=5;
}
```

1.可以fence一个ledger
2.也可以正常读取一个entry
3.如果带有previousLAC，则是一个longpoll操作，超时时间是请求里带的超时时间，请求的超时计算使用的是netty自带的hash时间轮

在读取entry时，ledger storage先从write cache内查找要读取的entry，然后再查找read cache内，最后没有的话会去读取磁盘，将读取到entry添加到read cache 内，并额外顺序读取额外的一批entry到read cache内，此处并未交由异步线程去读取，可能会影响真正需要的entry的读取速度。

### readCache的实现
read cache的默认大小是jvm可用堆外内存的1/4，额外预读entry大小不会超过read cache的一半，这也是防止一次预读不会占用太多空间。read cache内部也是将可用内存划分为几段来使用，具体优势跟write cache一样，它是一个环形存储，默认使用新的覆盖旧的缓存，这也是因为消息队列的特点，消息顺序消费，默认先淘汰旧的entry。


## ledger的清理
bookkeeper本身是不带entry清理功能的，你只能主动删除ledger，删除ledger接口内部只会删除保存在zk上ledger的metadata数据，bookkeeper内部有一个GarbageCollector，会定期删除entry log内被删除ledger的实际数据。相对于pulsar来说，pulsar本身增加了队列的保存期限语义，队列被删除、或者保存超期，pulsar会主动调用bookeeper删除ledger。

# 思考
1. Pulsar声称自己是原生的存算分离架构，不同于kafka的存储+计算，其实这么一看，也是不得已而为之，因为bookkeeper和pulsar两个组件并不是一起发明的，pulsar只是借用了bookkeeper做存储，自己抽象出了一些消息队列的概念，拼装起来成为一个消息队列，其实质存储实际是靠bookkeeper来完成的。

2. bookkeeper借鉴了LSM tree架构，journal相当于write ahead log， write/read cache类似与借鉴了MemTable，而entry log类似与SSTable，这种结构天然是为了高效写入数据而生。journal和entry log文件的写入，都是大块数据的写入，多个队列掺杂在一起，文件不会因为队列增多，队列partition增多而放大，出现像kafka一样的大量碎片文件，从而降低了集群的写入量。虽然bookkeeper建议双盘配置，journal使用ssd磁盘，entry log使用另外一块磁盘，实际entry log的写入也会反压单机的写入速度，即便journal这时有冗余的iops，也是无可奈何，没办法接收更多的写入请求。

3. journal文件写入使用了group commit来硬刷磁盘， entry log文件的写入轻度使用了page cache，在flush时仍然会被异步刷到磁盘上。不像kafka一样，完全使用page cache，其实这也是kafka无奈之举，天量的topic和partition产生的replica文件的写入，如果都去靠磁盘flush来保障持久性，磁盘IO性能势必会直线下降，只能避免主动flush，交由操作系统的page cache 自动来管理，但page cache的不可控性，大量小碎文件带来的磁盘碎片，多至数万个replica的复制，即便是SSD磁盘组成的集群，性能也会直线下降。

4. 一致性方面，bookkeeper自身只是一个无脑存储单元，ledger的副本，都是靠bookkeeper的客户端来完成，由客户端按照bookkeeper集群当前的数量，以及客户端的配置，临时生成ensemble，来完成多副本的保障，所以一致性，持久性都是靠客户端来完成，中间主要靠pulsar这个客户端来管理。而Kafka是靠自身的多副本主动复制机制来完成的，服务端自己来完成。相对来说，kafka的一致性和持久性会更统一一些，逻辑都在一块，而pulsar比较分散，需要客户端来主导，bookkeeper服务端来配合完成。

5. 上边说到bookkeeper是一个有限数据流，当单机宕机后，客户端会主动再已启动一个预设定数量的ensemble，继续完成消息数据的而写入，而且副本数量没有少，比如EM设定3，WQ设定3，AQ设定2，副本数量是3个，当一台机器宕机后，如果集群有冗余，客户端可以快速形成一个新的同等ensemble，因为它的数据流是分段形成的，新的段(ledger)不必一直设定在原有的机器上，可以很快在替代机器上生成。
而kafka不行，因为是无限数据流，集群宕机一台后，副本数量少一个，不能很快形成一个新的替代replica，必须替换机器，重新reassign partition leader和follower，来搬运历史数据，相对来说速度比较慢，会有一个较长的副本缺少的空窗期。
