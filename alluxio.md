         Alluxio简单分析
1、Alluxio介绍

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/alluxio架构.png)
 
如上图alluxio是一个以内存为中心的虚拟分布式存储系统。该系统介于计算框架和现有存储系统中间，担负着提升性能，统一计算框架透明访问异构存储功能。该分布式存储软件部署在计算和存储中间，为大数据的软件栈带来了显著的性能提升。
他的出现是为了解决传统大数据分析系统中的问题
1.	通常情况下在读数据领域，底层都是分布式存储，如HDFS。而应用层则是一些分布式计算框架，比如spark,mapreduce等。这些分布式框架往往直接大量的读取下层分布式存储数据，导致效率底下，性能损耗严重。
2.	计算进程异常Crush导致其缓存的数据丢失。
Alluxio将数据load在Alluxio内存中且上层计算框架共享这些数据，这样减少了本身软件栈读下层磁盘系统的IO时延。从而使访问下层磁盘系统的性能接近内存访问，显著提升应用性能。同时由于数据是隔离在的Alluxio内存中，即便计算框架的进程crush，数据还在Alluxio内存中，仍然不会丢失，这里不丢失是指的节点不故障而是计算进程curash。
Alluxio现在已经逐步演变为一个通用的分布式存储系统，将不同的计算框架和存储系统连接起来了。目前支持的计算框架：如Apache Spark，Apache MapReduce，Apache Flink，存储系统支持：如Amazon S3，OpenStack Swift，GlusterFS，HDFS， Ceph，OSS。
2、Alluxio的发展
- 	2012年诞生于UC Berkeley AMPLab，此前这个实验室孵化了Apache Mesos和Apache Spark等著名开源项目。
- 	2013年4月开源，现在由最初的Tachyon改名为Alluxio，基于Apache License 2.0开源标准，最新版本为Version 1.0 (Feb 23rd, 2016)。
- 	目前已经开发到1.8版本
- 	目前开源，活跃度非常高。自2013年4月开源以来，已有超过50个组织机构的900多贡献者参 与到Alluxio的开发中。包括阿里巴巴，Alluxio，百度，卡内基梅隆 大学，IBM，Intel，南京大学，Red Hat，UC Berkeley和Yahoo。
 
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/alluxio活跃度.png)

3、与Redis，Memcached key-value缓存的的区别
1.	Alluxio可以同时管理多个底层文件系统，将不同的文件系统统一在同一个名称空间下，让上层客户端可以自由访问统一名称空间内的不同路径，不同存储系统的数据。
2.	Alluxio提供文件接口，并存储且维护文件的metadata（比如记录文件分成哪几个block， 每一个block在哪台server上）。并提供fault tolerance的metadata服务。
3.	而Redis/Memcached为Nosql的key-value分布式缓存，并不提供文件接口。

4、整体架构

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/alluxio模块架构.png) 


如上图，整个系统分为三个大的模块，master，client和worker。
- 	master管理整个系统的元数据，以及和所有worker保持心跳，维护整个系统的健康状态。Master可以单个可以多个( 高可用)，多个情况下通过zookeeper协议保证元数据一致性以及自身的可靠性。
- 	worker负责本地memory管理及读写操作。Worker内部有cache机制，读取的热数据将长期驻存在内存。也支持将写入worker memory的数据定时checkpoint到底层存储系统。
- 	client：提供给应用访问alluxio的统一访问接口。读写过程中需要向master查询mate信息。如果文件数据在alluxio内存中，则直接本地或者远地内存读取，否则需要直接读写下层存储系统。
- 	三个组建都可以单独部署。

5、Alluxio文件组织

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/alluxio文件组织.png)

- 	如上图，存在master中的元数据为文件的元数据，在alluxio中文件以inode tree方式存放。文件属性包含id、名称、长度、创建/修改时间等，同时还包含了该文件包含的块信息和备份路径等。
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/alluxio文件切分.png)
- 	用户文件按照block组织。如上，文件对应的块信息保存在master上。块的用户数据信息则存放在worker中。
- 	worker中以block块粒度进行存储和管理。
- 	异构存储中则又是按照文件的粒度进行数据的存储和管理。

6、Alluxio读写

6.1、读分几种模式：
- 	在CACHE_PROMOTE(默认)模式下，如果读取的数据块在Worker上时，该数据块被移动到Worker的最高层。如果 该数据块不在本地Worker中，那么就将一个副本添加到本地Worker中，也就是worker的内存中。
- 	在cache模式下：如果该数据块不在本地Worker中，那么就将一个副本添加到本地Worker中。
- 	在no_cache模式下，不会创建副本，直接从下层存储系统读取数据。

6.2、写也分几种模式
- 	在CACHE_THROUGH模式下：数据被同步地写入到Worker和底层存储系统。
- 	在MUST_CACHE (默认)下，数据被同步地写入到Worker，但不会写入底层存储系统
- 	在THROUGH 模式下，数据被同步地写入到底层存储系统，但不会写入Worker。 
- 	在ASYNC_THROUGH 模式下，数据被同步地写入到Worker，并异步地写入底层存储系统。

6.3、写worker策略
- 	在LocalFirstPolicy (默认) 模式下：首先尝试使用本地Worker，如果本地Worker没有足够的容量容纳一个 数据块，那么就从有效的Worker列表中随机选择一个Worker。 
- 	在MostAvailableFirstPolicy模式下：使用拥有最多可用容量的Worker。 
- 	在RoundRobinPolicy 模式下：以循环的方式选取存储下一个数据块的Worker，如果该Worker没有足 够的容量容纳一个数据块，就将其跳过 
- 	在SpecificHostPolicy模式下：返回指定主机名的Worker。

7、Alluxio的容错机制
- 	master支持通过使用ZooKeeper启动多个Master。Master元数据的一致性通过记录日志journal的方式解决。
- 	worker的容错通过master和worker之间的心跳，worker失效的话自动拉起。
- 	然后worker对内存的数据持久化可以通过定时checkpoint的方式将alluxio的内存数据定时持久化到下层存储系统。
- 	在大数据spark场景可以对应用写入的数据进行世袭关系(lineage)处理(该方法是大数据计算常用的方法)，在计算框架crush时候可以通过记录的lineage数据之间的关系重新计算数据，从而恢复数据。这样可以减少因为要保证数据可靠性导致的数据写三副本的方式( 会占用大量网络带宽和增加时延)。

8、总结：
通过对alluxio简单的分析，可以看到，alluxio真正的使用场景是大数据环境下解决数据中心异构存储访问的问题。同时可以做到统一应用API访问接口，而这些接口可以不关心下面异构存储系统的差异。同时由于基于内存的分布式存储系统，可以在上分析的基础上提升数量级的读写访问性能。所以初略看alluxio是一个分布式内存存储系统，但是和我自己理解的分布式内存系统差别还是有点大。
首先它不存在在内存级别的多副本设计解决数据的可靠性问题。而取而代之的数据通过本地worker单写到下层的存储层，存储层可以通过多副本的方式保证数据的可靠性。数据写数据并不多副本存储在alluxio层。数据读也不是从本地进行读取，而是需要查询meta信息，有可能从远端的节点内存读取。
简单讲alluxio是一个提供统一应用API接口，同时屏蔽下层存储系统差异，且在读场景下由于数据长期贮存在内存中，可以提升读性能的一个软件系统。按照以前做存储产品经验看，这个alluxio的软件更像是一个存储网关产品软件。可以解决异构存储的问题，如解决数据孤岛，解决不同存储系统之间的数据迁移。同时由于数据会load 到alluxio主机内存端，所以整个设计又可以做到IO读写性能的提升。与其说这是一个分布式的内存存储系统还不如说是一个解决大数据计算框架下针对不同下层存储系统的网关软件。

