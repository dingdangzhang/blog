                                    Ceph多副本数据一致性

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/ceph架构.png)
     Ceph是当前开源社区比较受欢迎的一个分布式存储系统，应用于很多生产环境。和传统基于master中心节点方式的分布式系统不同的是，ceph是基于crush算法的去中心化的一个产品，数据的放置不依赖于中心节点。Ceph的基本逻辑架构如上图。文件被切分成一个一个的object.每个object大小为4M。通过对object名字进行hash再取模的方法可以将不同的object映射到不同的pg上。PG是一个逻辑概念，管理着一堆object。通过对PG ID进行crush伪随机算法可以得到3个固定的osd。ceph对于数据的放置没有专门的master节点来管理位置布局，而是通过hash和crush随机算法，保证数据离散在整个后端集群,并且数据三副本放置，保证了数据的可靠性。

#### 2.1、Ceph数据放置策略
![数据放置](https://github.com/dingdangzhang/blog/blob/master/file_image/无标题.png) 
Ceph对数据的可靠性是通过三副本的方式进行保证的。如上图。Ceph的读写操作采用主从模型，客户端要读写数据时，只能向对象所对应的主副本节点发起请求。主节点在接受到写请求时，会同步的向从副本中写入数据。当所有的副本都写入完成后，主节点才会向客户端报告写入完成的信息。因此保证了主从节点数据的高度一致性。读取的时候，客户端只会向主副本节点发起读请求，并不会有类似于数据库中的读写分离的情况出现，这也是出于强一致性的考虑。
再如2节中提到的不同的对象通过hash算法聚集在不同的PG里面，再由PG ID通过crush算法算出来的三个数据的OSD是伪随机固定的值，因此ceph对数据三副本的管理其实是按照PG为单位存放在不同的OSD上来实现的。因此对PG内object对象的访问操作其实就是对PG的操作，每个操作会通过记录日志的方式记录下来，目的是有副本故障后通过日志来恢复之前的数据。每个PG管理自己的pglog。
#### 2.2、PGlog
![pglog](https://github.com/dingdangzhang/blog/blob/master/file_image/ceph_pglog.png)
   Ceph多副本数据一致性通过pglog来保证。如上图，pglog记录该PG内所有对对象操作的记录(通常保存几千条记录)，比如修改，删除对象等。pglog每一条记录中记录了对对象的操作和版本信息。Ceph对pg内的每次操作都通过一条带版本的记录来表示，版本包括一个(epoch，version)来组成：其中epoch是osdmap(集群map)的版本，每当有OSD状态变化如增加删除等时，epoch就递增。version是PG内每次更新操作的版本号，递增的，由PG内的Primary OSD进行分配的。每一个副本都维护一个pglog.用于记录数据写入和OSD故障后的数据恢复,从而保证数据的一致性。
#### 2.3、Ceph三副本写流程
![rw](https://github.com/dingdangzhang/blog/blob/master/file_image/ceph_rw.png)
- 无论primary osd还是secondary osd数据的持久化都分两个阶段，一个写journal，一个写数据到filesystem paga cache,然后再通过定时flush将数据真正写盘。
- client将写请求发往primary osd, primary osd将请求和该操作生成的一条pglog序列化到事物中，记录改次操作。然后写journal。同时将请求和pglog发往其他两个OSD.
- primary osd上写完journal后会异步将这个journal的数据写到backend的文件系统上持久化数据。
- 副本OSD上收到请求后，会执行和primary OSD同样的操作，先写journal,再写数据。一旦journal写成功则返回primary OSD。primary OSD收到所有的secondary返回则返回client写成功。
#### 2.4、Ceph 感知集群状态
![heartbeat](https://github.com/dingdangzhang/blog/blob/master/file_image/ceph_heartbeat.png) 
- OSD之间有心跳，检测对方是否存在
- OSD和mon之间有心跳，OSD上报自己监控状态，也可以上报peer OSD健康状态
- 如果OSD主动下线，则会通知MON自己下线，如果是异常crush,那么其他OSD以及MON都能通过心跳得知该OSD下线
- 一旦有OSD下线，MON重新计算PG的primary OSD节点。并将结果通知给其他OSD节点。
- 此时这个OSD所在的PG处于降级状态，并且记录pglog, 等待恢复。
#### 2.5、PGlog参与故障恢复
ceph的故障恢复通俗讲就是通过对比各个副本的pglog.选出一个权威的pglog,并且通过pglog之间的差异，构建出一个missing object列表，然后恢复阶段根据这个missing列表来恢复对应的object。
Ceph 数据恢复在三副本恢复前期需要协调出一致的元数据，也就是权威的pglog,这个过程叫PG peering.该过程中该PG的IO会被阻塞，因为此时PG的三副本的pglog还不一致。只有peering完成后(三副本pglog一致)，再进行恢复数据阶段，IO才可以继续。
Ceph的三副本的恢复是基于pglog的一致性协议，具体恢复流程如下描述，故障场景可以分为两种场景：

##### 2.5.1、Primary OSD恢复
![primary_osd_restore](https://github.com/dingdangzhang/blog/blob/master/file_image/primary_osd_restore.png)
- 如上图，简单表示三副本写数据过程中的pglog操作示意图。
- last_complete: 在该指针之前的版本都已经在所有的OSD上完成更新
- last_updata: PG内最近一次更新的对象的版本，还没有在所有OSD上完成更新，在last_update与last_complete之间的操作表示该操作已在部分OSD上完成但是还没有全部完成
- 正常情况下，last_updata和last_complete都会指向三副本写成功后的最后一个操作。当有写期间有故障，会导致last_updata和last_complete和出现偏差，而这个偏差就是需要恢复的object。
- 当写过程中primary故障时候，由于副本还处于2副本状态，IO可以继续进行。此时mon会在剩下的osd中选择一个新主作为primary,继续在剩下的OSD内记录pglog.
- Primary OSD恢复时候，向之前的secondary OSD发送获取pglog请求。Primary OSD通过对比secondary OSD返回的PGlog信息得到自己的pglog已经落后于当前版本了。因此该OSD合并这个pglog,并找出missing object。供后续恢复流程异步重构。
- 一旦三个副本的pglog同步成同步的一致状态，整个PG恢复IO。同时异步进行数据重构。

##### 2.5.2、Secondary OSD恢复
![second_osd_restore](https://github.com/dingdangzhang/blog/blob/master/file_image/second_osd_restore.png) 
- secondary osd crush.由于还存在两副本，PG状态降级，继续提供IO。并且分别记录pglog
- secondary osd恢复，重新加入. Primary 发起peering操作，分别拉取secondary的pglog.对比pglog，确定加入的OSD上有peer_missing object。同步权威pglog(merge pglog)给该OSD副本，三副本状态恢复正常。
- Primary根据peer_missing object列表后台启动数据重构操作。使三副本数据最终达成一致。
##### 2.5.3、OSD永久故障
在临时故障也就是在很短时间恢复的故障，ceph是通过pglog来恢复数据,且pglog记录的log是有上限(万条记录)的，超过了则不能使用pglog恢复数据了。如对于永久故障，比如盘坏了，很长一段时间OSD不会恢复。那么pglog就不能起到恢复数据的作用了。这个时候ceph提供全量数据拷贝(backfill)方式.当新OSD加入当时候，mon会将该OSD加入PG组，由于该OSD没有任何数据，所以需要进行全部数据的拷贝。
如果新加入的OSD是拥有primary PG角色。那么在peering阶段发现不能通过pglog恢复。此时需要告诉MON自己需要全量复制。然后通过副本数据全量恢复该OSD。
如果新加入的OSD拥有secondary PG角色. 则接入PG组时候，primary自动发起全量同步将数据backfill到新加入的OSD secondary角色。
##### 2.5.4、Ceph scrub故障检测
Ceph作为数据强一致性系统，ceph的OSD还会定期启动线程来扫描部分对象，通过与对象的其他副本进行数据比较发现数据是否一致。如果存在不一致，ceph会抛出故障，让用户人工或者自动去解决。

2.5.5、总结：
- PGLog是Ceph进行数据恢复的重要依据，但是记录的日志数量有限，所以在发生故障后，尽快让故障节点重新上线，尽量避免产生Backfill操作，能极大的缩短恢复时间。
- PG在Peering阶段会阻塞客户端IO

