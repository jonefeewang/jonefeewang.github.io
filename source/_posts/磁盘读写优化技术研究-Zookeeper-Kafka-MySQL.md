---
title: 磁盘读写优化技术研究-Zookeeper/Kafka/MySQL
tags:
  - 磁盘读写优化
  - kafka
  - zookeeper
  - mysql
  - 磁盘读写优化
description: 本文对linux下磁盘的读写优化技术做了一个搜底和总览，包括基本的文件读写知识，磁盘，page cache ，zeror copy，mmap等知识，还对zookeeper 、kafka 、mysql 在使用磁盘读写文件时的优化技术做了分析。
toc: true
categories: 磁盘读写优化            
date: 2020-03-15 09:58:57
---

:warning: 原创文章，转载请注明出处

## [](#摘要 "摘要")摘要

本文对linux下磁盘的读写优化技术做了一个搜底和总览，包括基本的文件读写知识，磁盘，page cache ，zeror copy，mmap等知识，还对zookeeper 、kafka 、mysql 在使用磁盘读写文件时的优化技术做了分析。

## [](#关键词 "关键词")关键词

磁盘读写, 文件系统, block, page cache, zero copy, mmap, zookeeper, kafka, mysql

## [](#基础知识 "基础知识")基础知识

### [](#1-文件系统 "1.文件系统")1.文件系统

Linux使用文件系统来管理磁盘，将此磁盘分为一个个“block“单元，如下图所示，文件内容被写进一段连续的block单元内，中间空出来的即是我们平常所说的“碎片空间(fragment)”。常见的文件系统包含ext/ext2/ext3/ext4，以及windows使用NTFS等。

在现代ext4文件系统上，碎片空间已经不是问题，因为在分配每个文件存储的位置时，系统会特意让两个文件之间距离较远的位置，这样既方便文件不断变大，也不需要去整理碎片，但是极端场景下，比如整块磁盘空间都快用完的时候，就需要去挪动或迁移碎片，腾出可用空间，正常情况下是不需要的。

![](/2020/03/15/%E7%A3%81%E7%9B%98%E8%AF%BB%E5%86%99%E4%BC%98%E5%8C%96%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6-Zookeeper-Kafka-MySQL/file_blocks.svg "file blocks")

block内除了存储文件的真实内容之外，还需要存储每个文件的属性信息，  
比如gid/uid/permission/atime/modtime，以及文件存储在哪些block上的信息等元数据， 这类数据称之为metadata，在linux操作系统中叫index node，简称inode。ext3之前的文件系统，indoe需要使用一个集合来存储某个文件写在了哪些block上，随着文件的增大，这个集合会越来越大，造成系统的一个瓶颈。ext3之后的inode设计上，不再记录block集合信息，而是采用记录开始位置，和文件大小的两个属性，如下所示 ：

```
struct ext3_extent {  
 __le32  ee_block; /\* first logical block extent covers 文件的开始block信息 \*/   
 __le16  ee_len;   /\* number of blocks covered by extent 文件的大小\*/  
 __le16  ee_start_hi;  /\* high 16 bits of physical block \*/  
 __le32 ee_start; /\* low 32 bits of physical block \*/  
};  

```
### [](#2-数据落盘 "2. 数据落盘?")2\. 数据落盘?

由上边可以看出，读、写文件时，除了写文件内容本身之外，还要至少一次磁盘寻址，来写metadata一类的信息，总结这些额外的信息包括下边几个:

1.  文件所占的block数量
2.  文件最后一次访问时间atime
3.  文件最后一次的修改时间

针对这些额外的inode信息写操作，linux写文件时也有相应的两个函数来处理:

1.  fsync() : 将文件内容+文件metadata数据同步刷到磁盘上
2.  fdatasync(): 只将文件内容同步刷到磁盘上

java中的FileChannel api也提供了这个选项

```
/*
 @param metdata 是否同时将metadata信息同步flush到磁盘上  
*/  
public abstract void force(boolean metaData) throws IOException  
```
因此，在高频次写文件操作时，可以使用fdatasync减少一次磁盘操作，来提升写速度。

另外两个比较混淆的文件操作:

1.  FileDescriptor.sync() 底层同样调用的是fsync，将文件内容+文件metadata信息全部刷到了磁盘上
2.  OutputStream.flush只是将缓冲区内的数据发给操作系统，但有可能并未落盘，可能在pagecache内。

注意:  
OutputStream和FileoutputStream是一空函数,但 FilterOutputStream.flush  
,BufferedOutputStream.flush,ObjectOutputStream.flush确实是有内容的。

如果要把文件内容真正落盘，需要先调用stream的flush，将数据发送给操作系统，然后在调用FileDiscriptor的sync方法或 FileChannel.force方法，将数据真正落盘.

### [](#2-Zero-Copy技术 "2.Zero Copy技术")2.Zero Copy技术

在Java程序内通过InputStream和OutputStream读取或写入文件数据时，操作系统底层的实现一般是这样的：  

![](/2020/03/15/%E7%A3%81%E7%9B%98%E8%AF%BB%E5%86%99%E4%BC%98%E5%8C%96%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6-Zookeeper-Kafka-MySQL/read_write_file.svg "read and write a file")

1.  在用户空间内，程序发出读取数据指令。
2.  操作系统切换至内核空间，内核通过DMA读取磁盘数据。
3.  内核将自己读到的数据，从自己的buffer copy到用户的buffer内
4.  操作系统切换至用户空间
5.  继续用户处理逻辑
6.  完成处理后，将数据写入磁盘时，需要将buffer copy到内核的buffer内。
7.  操作系统切换至内核空间
8.  内核将数据通过disk controller写入磁盘
9.  操作系统切换至用户空间继续

可以看到，一个简单文件处理后保存操作，涉及到4次上下文切换和两次copy.

但linux系统都提供一种zero copy的技术支持，在用户空间内这个函数通常叫sendFile，可以使用linux man 命令(man sendfile)查看到，他的工作原理如下:

![](/2020/03/15/%E7%A3%81%E7%9B%98%E8%AF%BB%E5%86%99%E4%BC%98%E5%8C%96%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6-Zookeeper-Kafka-MySQL/zero_copy.svg "zero copy")

上图可见，用在程序里调用sendfile后，操作系统将数据从磁盘读入buffer后，直接发送给了网络，没有再把buffer拷贝到用户空间，但是仍然会有两次上下文切换。

nginx和apache httpd 服务器都支持sendfile指令。

具体到java中的“sendfile” 是FileChannel的transferto方法
``` 

public abstract long transferTo(long position,  
 long count,  
 WritableByteChannel target)  
 throws IOException  
// Transfers bytes from this channel's file to the given writable byte channel.  
```
### [](#3-mmap技术 "3.mmap技术")3.mmap技术

上边所说的sendfile使用zero copy技术，实际上我们并没有修改文件内容， 然后再保存，只是简单的将本次磁盘上的文件内容发送给了网络。

现代操作系统都支持一种技术叫mmap，将文件内容直接映射到用户空间内的一块内存buffer上，可以读也可以写，省去了kernel到用户空间的多次copy。

![](/2020/03/15/%E7%A3%81%E7%9B%98%E8%AF%BB%E5%86%99%E4%BC%98%E5%8C%96%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6-Zookeeper-Kafka-MySQL/mmap.svg "mmap")

这种技术能在内存中修改文件的内容，修改后还能保存，省去了buffer 在kernel和用户之间的copy，所以速度非常快。它实际上使用的类似OS管理虚拟内存的方式，将文件内容page in /page out。对应到java里的api 就是MappedByteBuffer，实际上也是一个byte buffer，同 directBuffer一样，这些内存都不在jvm的堆内。

使用MappedByteBuffer时，直接指定开始读的文件位置，和需要读多少长度的内容。

```

MappedByteBuffer out = file.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, length);  
```
但这种技术也是有缺点的：

1.  每个mmap生成的文件handle在一个进程内是有限制的，在linux下是64k，如果超过整个数字，就无法在map文件
2.  由于各种原因，jvm没有暴露关闭文件映射的方法，不能显式调用munmap()，要关闭掉这个文件映射，只能将MappedByteBuffer置为null，等java的gc过来回收，但如果内存很大，迟迟未gc的话，会耗尽文件handle。另外因为映射未及时关闭的话，不能再用普通的文件IO操作来读写文件。要显式关闭的话，只能使用hack的方法来调用sun.misc.Cleaner。
3.  对文件的所有修改都是在内存中，可以调用MappedByteBuffer的force方法，强制将内容flush到磁盘上，但是如果每次都这样强刷，是有性能损耗的。但如果让操作系统来管理什时候flush的话，机器突然宕机会有丢失数据的风险
4.  每次mmap文件时只能产生最大2GB的空间。

### [](#4-Page-Cache "4.Page Cache")4.Page Cache

在上边第二部分已经说过，数据不落盘的话，就是在操作系统的缓冲区内，这里就是pageCache，Linux操作系统默认在文件写入时，为了加快写入的速度，都会先写入page cache。page cache 实际是内存的缓冲区，在内存的分页Page内，写入数据的page称之为脏页(dirtyPage)，需要随后flush到磁盘上。

在读取文件数据时，操作系统读完之后，也会将数据暂存在page cache上，第二次读相同文件时，数据已经在内存page cache 内了，所以会更快一些。

Linux操作系统会将所有可用的内存作为page cache使用，物尽其用，不让其浪费，这就是在linux下看free memory时总是非常小的原因。

在数据写入page cache后，操作系统负载flush数据的线程会监视两个值，第一个是dirtypage 里的数据最长能存活多长时间必须刷入磁盘，第二个是flush线程多久需要 wakeup起来运行一次。flush线程运行时，除了看脏页存活时间外，还会看另外两个值，dirty_background_ratio和dirty_ratio.

这两个参数的意义：

1.  vm.dirty_background_ratio 是指dirtyPage占总内存百分比多少的时候，系统开始强制将dirtyPage的数据刷入磁盘。
2.  vm.dirty_ratio 指dirtyPage占据多少百分比内存时，开始强制block所有的IO操作，强制将dirtyPage内的数据刷入磁盘。

简单说，就是第一个控制dirtyPage可以有多大，但有可能第一个条件达到时， flush线程还未到达触发条件，第二控制是兜底，最大不能超过多大，否则所有的IO都暂停，强刷数据到磁盘上。

pageCache不能设置太大，也不能设置太小。太大的话，脏页里的数据会太多，刷入磁盘时卡顿时间会很长，太小的话，又起不到写数据是加速的作用。

## [](#ZooKeeper磁盘写优化 "ZooKeeper磁盘写优化")ZooKeeper磁盘写优化

### [](#1-groupCommit "1.groupCommit")1.groupCommit

在“一致性协议研究-zookeeper”章节中说过，zookeeper运行过程中，会生成两个文件，一个是txn-log，一个是snapShot文件。txn-log是事务日志，每个写请求都会转发给leader，leader发送propose数据给follower，flollower将数据落盘，然后ack 集群leader。Leader在收到足够的ack数量后，会发送commit请求给 follower，正式完成写请求。

如上一章所属，为了保证一致性，follower必须将数据“真正”落盘后，才能ack集群leader。如果在流量很大的情况下，这种频繁落盘操作势必会成为瓶颈，zookeeper怎么来解决这个性能问题呢？

follower在接收到leader的propose请求后，准确的说并未真实落盘，仍然在缓冲区内，zookeeper会缓存多条事务，等设定的阈值条数到了之后，再写入磁盘。相关代码如下:
```
// 插入新的事务，将事务序列化到文件中  
public synchronized boolean append(TxnHeader hdr, Record txn)  
 throws IOException{  
 //..  
 fos = new FileOutputStream(logFileWrite);  
 logStream=new BufferedOutputStream(fos);  
 oa = BinaryOutputArchive.getArchive(logStream);  
 FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);  
 fhdr.serialize(oa, "fileheader");  
 //...  
}  
  
//继续，插入新的事务  
 toFlush.add(si);  
 if (toFlush.size() > 1000) {  
 //积累到1000条后，开始flush，真正落盘  
 flush(toFlush);  
 }  
  
private void flush(LinkedList<Request> toFlush)  
 throws IOException, RequestProcessorException  
 {  
 //...  
 // 调用 zk database 的commit ，真正落盘开始  
 zks.getZKDatabase().commit();  
 //..  
 }  
   
public synchronized void commit() throws IOException {  
 //...  
 for(FileOutputStream log : streamsToFlush)  
 log.getChannel().force(false); //落盘  
 ///...  
 }  

```
以上代码摘自SyncRequestProcessor和FilTxnLog两个类。有代码可见，group commit其实就是batch commit.

### [](#2-文件预分配 "2.文件预分配")2.文件预分配

linux系统下，文件的一次写入最少会包含两次磁盘寻址，一次是写文件的真正内容，另外一次是更新文件的metadata信息，如上文所说的inode信息，需要更新使用的block数量，文件修改时间等信息。 zookeeper中的事务日志文件写入，采用了预分配的方法，每次创建一个新文件时，预先分配一定数量的文件空间(默认64M)(预定数量的block)，这样在平时高速追加文件内容时，就不需要每次去更新文件的meta信息，增加一次磁盘寻址了。相关的代码如下:

```

/*  
 \* Grows the file to the specified number of bytes. This only happenes if   
 \* the current file position is sufficiently close (less than 4K) to end of   
 \* file.   
 \*   
 \* @param f output stream to pad  
 \* @param currentSize application keeps track of the cuurent file size  
 \* @param preAllocSize how many bytes to pad  
 \* @return the new file size. It can be the same as currentSize if no  
 \* padding was done.  
 \* @throws IOException  
 */  
 public static long padLogFile(FileOutputStream f,long currentSize,  
 long preAllocSize) throws IOException{  
 long position = f.getChannel().position();  
 //判断当前的文件空间是否马上(差4kb)就要写满  
 if (position + 4096 >= currentSize) {  
 currentSize = currentSize + preAllocSize;  
 fill.position(0);  
 //预分配一个新的64M空间的文件  
 f.getChannel().write(fill, currentSize-fill.remaining());  
 }  
 return currentSize;  
 }  
```
### [](#3-省去更新metadata信息 "3.省去更新metadata信息")3.省去更新metadata信息

```

/*  
 * commit the logs. make sure that evertyhing hits the  
 * disk  
 */  
 public synchronized void commit() throws IOException {  
 if (logStream != null) {  
 logStream.flush();  
 }  
 for (FileOutputStream log : streamsToFlush) {  
 log.flush();  
 if (forceSync) {  
 long startSyncNS = System.nanoTime();  
 //FileChannel.force直写数据，不更新metadata，少一次磁盘寻道操作，优化写速度  
 log.getChannel().force(false);  

```
## [](#Mysql的磁盘写优化 "Mysql的磁盘写优化")Mysql的磁盘写优化

### [](#1-group-commit "1.group commit")1.group commit

mysql 中binlog的写入同样是每条事务的落盘，多个mysql 线程都同时在处理事务，怎么保障写binlog时不会冲突、乱序呢。这里边就需要一个锁(queue锁)，拿到锁的线程将事务写进一个统一的落盘事务queue，写入后释放锁，其他的thread可以继续写入新的事务。如下代码所示:

代码摘自:  
[https://github.com/percona/percona-server/blob/1b5dff5b9e5f8c797cfed966c73fbbf6d45cbd59/sql/log.cc](https://github.com/percona/percona-server/blob/1b5dff5b9e5f8c797cfed966c73fbbf6d45cbd59/sql/log.cc)

```
 //拿到queue锁  
mysql_mutex_lock(&LOCK_group_commit_queue);  
group_commit_entry \*orig_queue= group_commit_queue;  
entry->next= orig_queue;  
 //追加本次事务  
group_commit_queue= entry;  
DEBUG_SYNC(entry->thd, "commit_group_commit_queue");  
//释放锁  
mysql_mutex_unlock(&LOCK_group_commit_queue);  
```
如果这条线程发现自己是第一个初始化quque的，那么他自动变为group commit leader线程，他要负责将queue的内容最终落盘到binlog文件里去。
```

/*  
 The first in the queue handle group commit for all; the others just wait  
 to be signalled when group commit is done.  
 */  
 if (orig_queue != NULL)  
 entry->thd->wait_for_wakeup_ready();  
 else //我是group commit leader  
 trx_group_commit_leader(entry);  
```
leader线程会将queue里的内容先拷贝到自己的thread local queue里，然后再将自己thread local queue里的内容写进磁盘

落盘到binlog文件里的操作:

```

/*  
 Lock the LOCK_log(), and once we get it, collect any additional writes  
 that queued up while we were waiting.  
 */  
 //获取bin log文件锁  
 mysql_mutex_lock(&LOCK_log);  
 DEBUG_SYNC(leader->thd, "commit_after_get_LOCK_log");  
 //再次获取queue锁  
 mysql_mutex_lock(&LOCK_group_commit_queue);  
 //copy queue到thread local 里的queue里  
 current= group_commit_queue;  
 //清空原来的queue  
 group_commit_queue= NULL;  
 //释放queue锁  
 mysql_mutex_unlock(&LOCK_group_commit_queue);  

```
由上可以看出，mysql的事务写入也是批量的，通过中间的queue(mysql group commit queue)来实现批量的积攒，最终一次落盘操作。

## [](#Kafka的磁盘读写优化 "Kafka的磁盘读写优化")Kafka的磁盘读写优化

### [](#1-page-cache的使用 "1.page cache的使用")1.page cache的使用

Kafka主要用磁盘来存储消息，他对磁盘的使用优化技术用的很多，第一个就是pageCache的使用，kafka默认是不强刷磁盘的，所有的消息全部写入pageCache内，让操作系统来管理刷盘策略。但是kafka仍然提供了两个控制参数，多长时间需要刷一次盘，收到多少条消息后需要刷一次盘，如下代码：

```

 public static final String FLUSH_MESSAGES_INTERVAL_CONFIG = "flush.messages";  
 //注释中可以看出，不建议设置，让操作系统使用后台线程flush  
 public static final String FLUSH_MESSAGES_INTERVAL_DOC = "This setting allows specifying an interval at " +  
 "which we will force an fsync of data written to the log. For example if this was set to 1 " +  
 "we would fsync after every message; if it were 5 we would fsync after every five messages. " +  
 "In general we recommend you not set this and use replication for durability and allow the " +  
 "operating system's background flush capabilities as it is more efficient. This setting can " +  
 "be overridden on a per-topic basis (see <a href=\\"#topicconfigs\\">the per-topic configuration section</a>).";  
  
 public static final String FLUSH_MS_CONFIG = "flush.ms";  
 //注释中可以看出，不建议设置，让操作系统使用后台线程flush  
 public static final String FLUSH_MS_DOC = "This setting allows specifying a time interval at which we will " +  
 "force an fsync of data written to the log. For example if this was set to 1000 " +  
 "we would fsync after 1000 ms had passed. In general we recommend you not set " +  
 "this and use replication for durability and allow the operating system's background " +  
 "flush capabilities as it is more efficient.";  
   
  
 //到达配置时间后，强制刷盘   
if (unflushedMessages >= config.flushInterval)  
 flush()  
 //xxxx  
  
 /*  
 * Commit all written data to the physical disk  
 */  
 public void flush() throws IOException {  
 //强制刷盘，包括元数据也刷进磁盘  
 channel.force(true);  
 }  
```

### [](#2-zeroCopy技术的使用 "2. zeroCopy技术的使用")2\. zeroCopy技术的使用

kafka在消费者拉取消息时，需要将磁盘的数据发给消费者，这时就是用”sendFile”的zero copy概念，相关代码如下(截取自PlaintextTransportLayer.java):
```  

@Override  
public long transferFrom(FileChannel fileChannel, long position, long count) throws IOException {  
 //将文件内容写入网络socket，zero copy  
 return fileChannel.transferTo(position, count, socketChannel);  
}  

```
### [](#3-文件预分配 "3.文件预分配")3.文件预分配

(详细内容，参见上面zookeeper-文件预分配技术)，kafka记录消息的文件称之为segement，在创建这样的文件时也是用了 文件预分配技术优化，相关代码如下(截取自Log.scala):

```

  
if (logSegments.isEmpty) {  
 // no existing segments, create a new mutable segment beginning at logStartOffset  
 addSegment(LogSegment.open(dir = dir,  
 baseOffset = logStartOffset,  
 config,  
 time = time,  
 fileAlreadyExists = false,  
 initFileSize = this.initFileSize,  
 //预分配文件大小  
 preallocate = config.preallocate))  
}  
```

### [](#4-mmap技术 "4. mmap技术")4\. mmap技术

内存映射文件，如上边基础知识所述，通过内存来映射磁盘上的数据文件，达到较快的读写速度。kafka通过mmap来读写index文件，相关代码如下(截取自AbstractIndex.scala):
``` 

abstract class AbstractIndex\[K, V\](@volatile var file: File, val baseOffset: Long,  
 val maxIndexSize: Int = -1, val writable: Boolean) extends Closeable with Logging {  
​  
 // Length of the index file  
 @volatile  
 private var _length: Long = _  
 protected def entrySize: Int  
​  
 /*  
 Kafka mmaps index files into memory, and all the read / write operations of the index is through OS page cache. This  
 avoids blocked disk I/O in most cases.  
​  
 To the extent of our knowledge, all the modern operating systems use LRU policy or its variants to manage page  
 cache. Kafka always appends to the end of the index file, and almost all the index lookups (typically from in-sync  
 followers or consumers) are very close to the end of the index. So, the LRU cache replacement policy should work very  
 well with Kafka's index access pattern.  
 */  
   
 @volatile  
 protected var mmap: MappedByteBuffer = {  
 val newlyCreated = file.createNewFile()  
 //创建RandomAccessFile  
 val raf = if (writable) new RandomAccessFile(file, "rw") else new RandomAccessFile(file, "r")  
 
 //开始map 文件  
 if (writable)  
 raf.getChannel.map(FileChannel.MapMode.READ_WRITE, 0, _length)  
 else  
 raf.getChannel.map(FileChannel.MapMode.READ_ONLY, 0, _length)  
 
 }  

 ```

另外一个有趣的地方是kafka的munmap方法，实际上调用的是DirectByteBuffer的cleanr方法来关闭文件映射，使用方式比较hack。代码如下(截取自MappedByteBuffers.java):

```

private static MethodHandle unmapJava7Or8(MethodHandles.Lookup lookup) throws ReflectiveOperationException {  
 /* "Compile" a MethodHandle that is roughly equivalent to the following lambda:  
 *  
 * (ByteBuffer buffer) -> {  
 *   sun.misc.Cleaner cleaner = ((java.nio.DirectByteBuffer) byteBuffer).cleaner();  
 *   if (nonNull(cleaner))  
 *     cleaner.clean();  
 *   else  
 *     noop(cleaner); // the noop is needed because MethodHandles#guardWithTest always needs both if and else  
 * }  
 */  
 Class<?> directBufferClass = Class.forName("java.nio.DirectByteBuffer");  
 Method m = directBufferClass.getMethod("cleaner");  
 m.setAccessible(true);  
 MethodHandle directBufferCleanerMethod = lookup.unreflect(m);  
 Class<?> cleanerClass = directBufferCleanerMethod.type().returnType();  
 MethodHandle cleanMethod = lookup.findVirtual(cleanerClass, "clean", methodType(void.class));  
 MethodHandle nonNullTest = lookup.findStatic(MappedByteBuffers.class, "nonNull",  
 methodType(boolean.class, Object.class)).asType(methodType(boolean.class, cleanerClass));  
 MethodHandle noop = dropArguments(constant(Void.class, null).asType(methodType(void.class)), 0, cleanerClass);  
 MethodHandle unmapper = filterReturnValue(directBufferCleanerMethod, guardWithTest(nonNullTest, cleanMethod, noop))  
 .asType(methodType(void.class, ByteBuffer.class));  
 return unmapper;  
 }  
```
参考:

1.  nachoparker 2018 [https://ownyourbits.com/2018/05/02/understanding-disk-usage-in-linux/](https://ownyourbits.com/2018/05/02/understanding-disk-usage-in-linux/)
2.  Lokesh Gupta , [https://howtodoinjava.com/java/io/how-java-io-works-internally-at-lower-level/](https://howtodoinjava.com/java/io/how-java-io-works-internally-at-lower-level/)
3.  Shawn Xu, [https://medium.com/@xunnan.xu/its-all-about-buffers-zero-copy-mmap-and-java-nio-50f2a1bfc05c](https://medium.com/@xunnan.xu/its-all-about-buffers-zero-copy-mmap-and-java-nio-50f2a1bfc05c)
4.  [https://stackoverflow.com/questions/8263995/standardopenoption-sync-vs-standardopenoption-dsync](https://stackoverflow.com/questions/8263995/standardopenoption-sync-vs-standardopenoption-dsync)
5.  [https://stackoverflow.com/questions/4072878/i-o-concept-flush-vs-sync](https://stackoverflow.com/questions/4072878/i-o-concept-flush-vs-sync)