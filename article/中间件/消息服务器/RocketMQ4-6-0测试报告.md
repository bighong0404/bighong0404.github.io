# 1. 服务器(1.21)硬件信息

## 1.1 CPU, 内存, 网卡

IP: 192.168.1.21
CPU: Intel(R) Xeon(R) CPU E5-2609 v3 @ 1.90GHz, 12核
内存: 48G 2133 MHz
网卡: I350 Gigabit 千兆网卡, 1000Mb/s = 125MB/s, 网络传输瓶颈是125MB/s

## 1.2 磁盘写瓶颈
```
[root]# dd if=/dev/zero of=test.dd bs=1M count=20000
20000+0 records in
20000+0 records out
20971520000 bytes (21 GB) copied, 85.9679 s, 244 MB/s
```

## 1.3 磁盘读瓶颈
```
[root]# hdparm -tT --direct /dev/mapper/vg_devenv22-lv_home
/dev/mapper/vg_devenv22-lv_home:
Timing O_DIRECT cached reads: 1262 MB in 2.01 seconds = 627.28 MB/sec
Timing O_DIRECT disk reads: 390 MB in 3.00 seconds = 129.87 MB/sec
```

---

# 2. 单组服务器 – 异步提交模式

## 2.1 服务器部署情况

以1m-1s形式部署在1.21/1.20服务器,
同步模式: 同步双写, 刷盘模式: 异步刷盘

## 2.2 批量消息提交

使用send(Collection)批量发送的方法.

- coreSize=16线程池发送6w批1k条100Byte(共6000W)的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
1G	| 140~170%| 	45~55M	| 写入30M,同步40M| 15.6W	| 59621000	| 99.37%
4G	| 150~190%| 	45~55M	| 写入30M, 同步40M| 	16.3W	| 59705000	| 99.51%

- coreSize=32线程池发送6w批1k条100Byte(共6000W)的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
1G|	150~170%|	45~55M|	写入33M, 同步40M|	16.6W|	59688000|	98.87%|
4G|	160~190%|	45~55M|	写入33M, 同步40M|	15.6W|	59314000|	98.86%|

## 2.3 单条消息提交

### 2.3.1 带回调的单消息send方法

- coreSize=16线程池发送100w条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
4G|	前2s: 300%, 之后 2%|	0.5~0.8M| 写入几百k, 之后大约0|	日志看不出(一分钟统计一次)|	4963|0.5%

- coreSize=64线程池发送0.3W条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
4G|	前2s: 300%, 之后 2%|	前2s: 2~4M,之后几K|	前2s: 2~4M,之后大约0|	日志看不出(一分钟统计一次)|	14755|	49.2%|

### 2.3.2 同步等待响应的单消息send方法

- coreSize=16线程池发送100W条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
4G|	439 %|	4M|	写入4M,同步2.5M|	8222|	100W|	100%|


- coreSize=64线程池发送100W条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
4G|	552%|	6~7M|	写入9M,同步6M|	11488|	100W|	100%

### 2.3.3 不等待响应的单消息sendOneway方法

- coreSize=16线程池发送100W条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
4G|	559.7%|	5~7M|	写入21M,同步6M|	7370|	442704|	44.27%


- coreSize=64线程池发送200W批条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
4G| 	563.1%| 	5~7M| 	写入21M,同步6M| 	8625| 	886649| 	44.33%


- coreSize=64线程池发送30W批条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| ---| ---| ---| ---| :-:| ---
4G|	556.1%|	6~7M|	写入21M,同步6M|	日志看不出(一分钟统计一次)|	1251649|	41.72%


- coreSize=64线程池发送8W批条100Byte的消息.

broker堆内存分配	| CPU	| 磁盘写MB/s	| 网络流入	| TPS	| broker成功接收消息数	| 成功率
:-:| :-:|:-:|:-:|:-:|:-:| ---
4G|	-|	-|	-|	-|	34021|	42.52%

## 2.4 消息消费

消费者数量|	broker堆内存分配|	CPU|	磁盘读MB/s|	网络流出	|TPS
:-:| :-:|:-:|:-:|:-:|:-:
1|	1G|	65~75%|	85M左右|	85M左右|	34.7W
1|	4G|	55%左右|	接近100M| 接近100M|	36.3W
2|	1G|	45~55%|	80M左右|	40M * 2|	30.2W


## 2.5 结论

### **提交消息**

rocketmq的Java客户端SDK异步提交消息有单消息与批量消息的send重载方法, 不管send集合还是单条消息, 都会直接往服务端发数据. 并发请求次数一多, 可能会有timeout, overload异常. 另外批量消息send方法不支持callback回调.

1) 使用带回调的单消息send方法, 不适合大量放入线程池处理, 成功率很低. 因为消息提交到producer内部线程池时候会带入创建任务时间, 在发送前检测当前时间与任务时间的时间差是否超过设置的超时时间, 是的话直接返回超时, 并没有推去Broker.

2) 同步等待响应的单消息send方法比较稳定, broker端100%成功接收, 但偶尔会出现Broker响应超时情况, 不一定是接受失败.

3) 不等待响应的单消息send方法, 百万级别的消息成功接受率50%不到, 不建议使用.

### **消费消息**

消费消息的效率非常高, 一个消费者实例在不做业务只消费消息的情况下TPS有30W+, 能做到榨干千兆内网带宽.

---

# 3. 双组集群 – 异步提交模式

## 3.1 服务器部署情况

以官方推荐的集群最低规模2m-2s模式部署, 4个broker实例部署在不同服务器.
同步模式: 同步双写, 
刷盘模式: 异步刷盘

## 3.1 批量消息提交

- coreSize=32线程池发送8w批1k条100Byte(共8000W)的消息.

组别|brokerbroker堆内存分配	| CPU| 磁盘写MB/s	| 网络流入| TPS| broker成功接收消息数	| 成功率
:-:|:-:| ---| ---| ---| ---| :-:| ---
a(21)|4G|	132.45%|	30-50M|写入30+-5M,同步46+-5M|14.7W|39769000|99.42%
b(20)|4G|	140.5%|30-50M|写入30+-5M,同步46+-5M|14.1W|39855000|99.64%

- coreSize=64线程池发送8w批1k条100Byte(共12000W)的消息.

组别|brokerbroker堆内存分配	| CPU| 磁盘写MB/s	| 网络流入| TPS| broker成功接收消息数	| 成功率
:-:|:-:| ---| ---| ---| ---| :-:| ---
a(21)| 	4G| 	138.75%| 	30-50M| 	写入30+-5M,同步40+-5M| 	14.5W| 	59346000| 	98.91%
b(20)| 	4G| 	140.65%| 	30-50M| 	写入30+-5M,同步40+-5M| 	14.4W| 	59175000| 	98.63%


## 3.2 消息消费

消费者数量|	组别|	broker堆内存|	CPU|	磁盘读MB/s|	网络流出|	TPS
:-:|:-:| ---| ---| ---| ---| :-:|
1|	a(21)|	4G|	轮流消费, 不好观察|同左|同左|同左
1|	b(20)|	4G|	轮流消费, 不好观察|同左|同左|同左
2|	a(21)|	4G|	58.05%|	90M左右|	100M左右|	29.1W
2|	b(20)|	4G|	48.75%|	85M左右|	100M左右|	35.4W

## 3.3 结论

当集群中有多组主从, 消费者只有一个实例的时候, 会在broker之间轮流消费消息.

---

# 4 测试工具/命令

TPS: RocketMQ日志store.log提供, RocketMQ 控制台
网速: iftop -B
磁盘: iostat -xmd 1
cpu/内存: top -d 1 -n 20 | grep pid

# 5 最佳实践建议

## Broker部署
1. linux系统强烈建议先执行bin/os.sh脚本. 这是官方提供的调整内核参数的脚本. 不先执行会出现CPU占用飙升直至os crush.

2. 官方推荐的broker jvm参数:
```
-server -Xms8g -Xmx8g -Xmn4g
-XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0
-XX:SurvivorRatio=8 -XX:+DisableExplicitGC
-verbose:gc -Xloggc:/dev/shm/mq_gc_%p.log -XX:+PrintGCDetails
-XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime
-XX:+PrintAdaptiveSizePolicy
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m
```
3. broker集群模式, 官方建议最低配2m-2s. 在4.5.0版本之后新增了” 自动容灾切换模式DLedger”, 区别是主从模式下, 若主挂了, 不会选举一个主Broker出来. 而自动容灾切换DLedger模式支持, 但是该模式不支持批量发消息. 功能未完善, 暂不推荐使用.

4. 线上关闭topic自动创建, 通过控制台手动创建, 防止负载失效

5. Broker主从同步方式支持同步双写以及异步复制, 对数据精确度不苛刻可以配置异步复制, 但这是Broker级别的配置, 需要重启服务器.

## producer使用
1. 数量级高于万级并且对可靠性要求不高的消息, 推荐使用批量发送接口, 或者使用吞吐更高的kafka.
2. 数量级低于万级并且对可靠性有要求的消息, 建议使用同步等待响应的单发接口, 强烈不建议使用不等待响应的sendOneway()接口.
3. dubbo服务化建议
4.  由于业务强相关, 不好提供带回调send方法以及事务消息方法, 考虑封装SDK给业务系统调用
5.  提供同步单消息send方法, 批量消息send方法

---

# 6 遇到的问题

## 6.1 吞吐问题, TIMEOUT, OVERLOAD等

该问题根源是Broker负载过高的问题. 处理方式分两种
**Broker端:**
- 提高主Broker硬件配置(内存, 磁盘性能), 增加主-从实例来分流
- 参考https://github.com/apache/rocketmq/issues/1585

**Producer端:**
- 若是使用单条消息提交方式, 使用同步等待响应结果的send消息
- 控制发送频率

## 6.2 CPU飙升, 服务器假死

**服务器配置相关**

- broker内存分配 –Xmx1g –Xms1g
- broker主从同步双写, 异步刷盘
- 系统swap区默认开启, 分配15G
- 系统内核版本 Linux 2.6.32-431.el6.x86_64

**测试行为**
- 通过coreSize=16的pool往broker连续写入40000批1000条100Byte消息.

**症状**
- 客户端很多以下异常
  - MQBrokerException: CODE: 2 DESC: [TIMEOUT_CLEAN_QUEUE]broker busy, start flow control for a while, period in queue: 507ms, size of queue: 9
  -  MQBrokerException: CODE: 2 DESC: [PCBUSY_CLEAN_QUEUE]broker busy, start flow control for a while, period in queue: 461ms, size of queue: 13
  -  MQBrokerException: CODE: 2 DESC: [REJECTREQUEST]system busy, start flow control for a while

- 服务器卡死, 无响应

**原因分析:**
    运维同学帮忙看到是swap区io阻塞, 进而CPU占用飙升, 最后服务器假死无响应.

**处理方案:**
    限制swap区的分配. swap区是在内存昂贵的年代为了给系统腾出足够的物理内存而划出来的基于磁盘的虚拟内存, 以缓解内存不足的情况. 跟运维了解到阿里云服务器默认关闭swap区. 并且也找到官方说明, 建议限制swap区.

  官方提供脚本bin/os.sh, 调整了一些系统内核参数(包含限制swap区), 强烈建议执行.
- [os.sh内容](http://blog.soliloquize.org/2018/09/02/RocketMQ-ossh%E5%8F%82%E6%95%B0%E8%AF%B4%E6%98%8E/)

```properties
这个参数应该也是用来控制空闲内存大小的。但是很多发行版没有这个参数，一般根据部署环境看是否支持，不支持的话不进行设置应该也没关系。
### sudo sysctl -w vm.extra_free_kbytes=2000000

设置系统需要保留的最小内存大小。当系统内存小于该数值时，则不再进行内存分配。
这个命令默认被注释掉，根据文档看，如果设置了不恰当的值，比如比实际内存大，则系统可能直接就会崩溃掉。
实际数值应该根据RocketMQ部署机器的内存进行计算，经验数值大概是机器内存的5% - 10%。
这个数值设置的过高，则内存浪费。若设置的过低，那么在内存消耗将近时，RocketMQ的Page Cache写入操作可能会很慢，导致服务不可用。
### sudo sysctl -w vm.min_free_kbytes=1000000

控制是否允许内存overcommit。设为1，则是允许。当应用申请内存时，系统都会认为存在足够的内存，准许申请。
### sudo sysctl -w vm.overcommit_memory=1

设置为1会释放page cache。这个操作是一次性的，在执行该命令时释放page cache，没有持续性作用。
### sudo sysctl -w vm.drop_caches=1

该配置用于控制zone内存使用完之后的处理策略，设为0则禁止zone reclaim。对于大量使用缓存的应用来说，一般都需要禁止掉。
### sudo sysctl -w vm.zone_reclaim_mode=0

Rocket使用MMAP映射文件到内存，因此需要设置映射文件的数量，避免MMAP操作失败。
### sudo sysctl -w vm.max_map_count=655360

设置内存中的脏数据占比，当超过该值时，内核后台线程将数据脏数据刷入磁盘。
### sudo sysctl -w vm.dirty_background_ratio=50

设置内存的脏数据占比，当超过该至时，应用进程会主动将数据刷入磁盘。
### sudo sysctl -w vm.dirty_ratio=50

内核线程会定期启动将内存中的旧数据刷入磁盘，通过此参数可以控制启动间隔。
### sudo sysctl -w vm.dirty_writeback_centisecs=360000

设置一次读操作会加载几个数据页。
### sudo sysctl -w vm.page-cluster=3

vm.swappiness用于控制使用系统swap空间比例，一般设置为0是禁止使用swap，设为1估计是某些系统不支持设置为0.
RocketMQ的数据存储基于MMAP，大量使用内存，因此需要禁止swap，否则性能会急剧下降。
### sudo sysctl -w vm.swappiness=1

设置可以打开的文件符号数。RocketMQ的存储都是基于文件的，因此稍稍设大默认值。
### echo 'ulimit -n 655350' >> /etc/profile

设置一个进程能够打开的文件数。
### echo '* hard nofile 655350' >> /etc/security/limits.conf

RocketMQ使用MMAP映射文件到内存，在存在新消息的时候都是追加顺序写，投递消息的时候则是根据Offset从CommitLog进行随机读取，
Deadline调度方法会在调度时间内合并随机读为书序读，因此对RocketMQ性能有帮
### echo 'deadline' > /sys/block/${DISK}/queue/scheduler
```

---

# 7. Broker配置

```properties

#nameServer地址，分号分割
namesrvAddr=ip1:9876;ip2:9876
#集群名称
brokerClusterName=rocketmq-cluster
#主从一致
brokerName=broker-a
#0表示Master，>0表示Slave, Master-Slave模式适用
brokerId=0
#Broker 对外服务的监听端口
#线上环境, 主从端口保持一致
listenPort=10911
#当前broker监听的IP
brokerIP1=主网卡1ip
#存在broker主从时，broker从节点会连接主节点配置的brokerIP2来同步
#默认不配置brokerIP1和brokerIP2时，都会根据当前网卡选择一个IP使用，
#当你的机器有多块网卡时，很有可能会有问题
brokerIP2=网卡2ip
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#从节点是否可读, 默认false
slaveReadEnable=true
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=false
#是否允许 Broker 自动创建订阅组
autoCreateSubscriptionGroup=true
#支持消息轨迹
traceTopicEnable=true
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=48
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#在发送线程池任务队列的最大等待时间, 超过时间任务会被移除, 默认200ms
waitTimeMillsInSendQueue=500
#服务端发送线程个数，建议配置成Cpu核数
sendMessageThreadPoolNums=12
#消息存储到commitlog文件时获取锁类型，如果为true使用ReentrantLock否则使用自旋锁
useReentrantLockWhenPutMessage=false
#一次服务端消息拉取,消息在内存中传输允许的最大传输字节数默认262144Byte
maxTransferBytesOnMessageInMemory=262144
#一次服务消息拉取,消息在内存中传输运行的最大消息条数,默认为32条
maxTransferCountOnMessageInMemory=2000
#一次服务消息端消息拉取,消息在磁盘中传输允许的最大字节, 默认65536Byte
maxTransferBytesOnMessageInDisk=65536
#一次消息服务端消息拉取,消息在磁盘中传输允许的最大条数,默认为8条
maxTransferCountOnMessageInDisk=2000
#磁盘空间最大利用率, 默认75%
diskMaxUsedSpaceRatio=75
#磁盘空间警戒水位，超过，则停止接收新消息（出于保护自身目的）默认是90
diskSpaceWarningLevelRatio=90
#磁盘空间强制删除文件水位。默认是85
diskSpaceCleanForciblyRatio=85
#存储路径
storePathRootDir=/home/rocketmq/rocketmq-master-a/store
#commitLog 存储路径
storePathCommitLog=/home/rocketmq/rocketmq-master-a/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/home/rocketmq/rocketmq-master-a/store/consumequeue
#消息索引存储路径
storePathIndex=/home/rocketmq/rocketmq-master-a/store/index
#checkpoint 文件存储路径
storeCheckpoint=/home/rocketmq/rocketmq-master-a/store/checkpoint
#abort 文件存储路径
abortFile=/home/rocketmq/rocketmq-master-a/store/abort
#允许的最大消息体, 默认4M
maxMessageSize=6291456

```
---

# 8. 与Kafka的对比

翻译自: [http://rocketmq.apache.org/docs/motivation/](http://rocketmq.apache.org/docs/motivation/)

![](https://upload-images.jianshu.io/upload_images/4039130-69f049c41925a07d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 8.1 RocketMQ比Kafka优势的特性

- 不需要zookeeper, 内置name server
- 多种方式来确保消息可靠性(消息事务, 主从同步双写, 同步刷盘, 消息保存返回确认, 发送失败重试, 消费失败重推等)
- 支持延时消息,特定的level(延时5s，10s，1m等18个预设值), 不支持自定义时间精度
- 支持消息打tag, 方便服务器过滤使用, consumer拉取消息可以根据tag过滤, 相当于消息二级分类
- 支持消息绑keys, 代表这条消息的业务关键词，服务器会根据keys创建哈希索引，设置后，可以在Console系统根据Topic、Keys来查询消息，
- 控制台展示数据细粒度更小, 支持显示消息内容, 消息轨迹等
- 在多Topic场景下, RocketMQ表现稳定, 而kafka吞吐急剧下降

## 8.2 Kafka比RocketMQ优秀的特性

- 机器配置需求低, 特别是内存, RockeMQ默认要求4G, 官方推荐8G, 而Kafka默认2G
- 低Topic场景下Kafka吞吐远超RocketMQ
- RocketMq手动配置broker的master或slave角色(最低2m-2s-async), 因此master挂了不会通过选举一个出来, 需要人工处理.

---

# 附录

[Apache RocketMQ官网]( http://rocketmq.apache.org/docs/quick-start/)

[Apache RocketMQ开发者指南(包含介绍以及部署)](https://github.com/apache/rocketmq/tree/master/docs/cn)

[阿里中间件团队博客, 10分钟入门]( http://jm.taobao.org/2017/01/12/rocketmq-quick-start-in-10-minutes/)

[阿里RocketMQ迈入50万TPS消息俱乐部](http://jm.taobao.org/2017/03/23/20170323)

[阿里中间件团队博客, Kafka vs RocketMQ - Topic数量对单机性能的影响](http://jm.taobao.org/2016/04/07/kafka-vs-rocketmq-topic-amout/)

  **集群部署参考**

 https://www.jianshu.com/p/9f1b8d2f73dd

 https://blog.csdn.net/leexide/article/details/80035470

**RocketMQ4.3.x 配置**

 https://www.cnblogs.com/zhyg/p/10255656.html

**rocketmq拓展程序**

 https://github.com/apache/rocketmq-externals

**console后台管理系统**

 https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console

 https://github.com/apache/rocketmq-externals/blob/master/rocketmq-console/doc/1_0_0/UserGuide_CN.md

  **系统安装参考**

 https://www.cnblogs.com/immense/p/11406399.html

**RocketMQ消息存储实现**

(一)https://www.jianshu.com/p/b73fdd893f98
(二)https://www.jianshu.com/p/6d0c118c17de