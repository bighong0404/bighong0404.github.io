> https://juejin.cn/book/6844733792683458573



# 1. 日志存储



##　日志根目录

 检查点文件:

- **cleaner-offset-checkpoint**: 清理检查点文件，用来记录每个主题的每个分区中已清理的偏移量。

- **log-start-offset-checkpoint**: 日志起始点文件.  各个副本在变动 LEO 和 HW 的过程中，logStartOffset 也有可能随之而动。Kafka也有一个定时任务来负责将所有分区的 **logStartOffset** 写到起始点文件中，定时周期由 broker 端参数 `log.flush.start.offset.checkpoint.interval.ms` 来配置，默认值为60000。
- **recovery-point-offset-checkpoint**:  恢复点文件. Kafka 中会有一个定时任务负责将所有分区的 **LEO** 刷写到恢复点文件中，定时周期由broker端参数 `log.flush.offset.checkpoint.interval.ms` 来配置，默认值为60000.
- **replication-offset-checkpoint** : 复制点文件.  Kafka 有一个定时任务负责将所有分区的 **HW** 刷写到复制点文件中，定时周期由 broker 端参数 `replica.high.watermark.checkpoint.interval.ms` 来配置，默认值为5000。





meta.properties







## topic目录文件

![image-20210831154017647](img/image-20210831154017647.png)

日志是一<topic>-<分区数>来命名目录. 

top-分区目录的文件如下

```shell
-rw-rw-r-- 1 root root 10485760 8月  27 09:16 00000000000000018913.index
-rw-rw-r-- 1 root root     3963 8月  27 10:01 00000000000000018913.log
-rw-rw-r-- 1 root root       10 8月  24 14:22 00000000000000018913.snapshot
-rw-rw-r-- 1 root root 10485756 8月  24 14:27 00000000000000018913.timeindex
-rw-rw-r-- 1 root root       12 8月  24 14:22 leader-epoch-checkpoint
```

日志数据文件

- log文件: 日志文件
- index文件: 偏移量索引文件
- timeindex文件: 时间戳索引文件

辅助文件

- deleted:  标识文件可以删除, 由"delete-file"延迟任务去执行删除
- cleaned
- swap”等临时文件
- snapshot
- txnindex
- leader-epoch-checkpoint:  分区leader纪元检查点. 保存了每一任leader开始写入消息时的offset, 会有线程定时更新. 用于处理lead变更时候可能产生的消息数据不一致问题.

每个 LogSegment 都有一个`基准偏移量 baseOffset`，用来表示`当前 LogSegment 中第一条消息的 offset`。

基准偏移量是一个64位的长整型数，**日志文件和两个索引文件都是根据基准偏移量（baseOffset）命名**的，名称固定为20位数字，没有达到的位数则用0填充。



# 2. 日志格式



// todo...



# 3. 日志索引



## 3.1 日志索引方式

Kafka 中的索引文件以`稀疏索引（sparse index, 跳表原理）`的方式构造消息的索引，它并不保证每个消息在索引文件中都有对应的索引项。

每当写入一定量（由 broker 端参数 log.index.interval.bytes 指定，默认值为4096，即 4KB）的消息时，偏移量索引文件和时间戳索引文件分别增加一个偏移量索引项和时间戳索引项，增大或减小 log.index.interval.bytes 的值，对应地可以增加或缩小索引项的密度。

稀疏索引通过 MappedByteBuffer 将索引文件映射到内存中，以加快索引的查询速度。偏移量索引文件中的偏移量是单调递增的，查询指定偏移量时，使用二分查找法来快速定位偏移量的位置，如果指定的偏移量不在索引文件中，则会返回小于指定偏移量的最大偏移量。



## 3.2 日志文件切分策略

日志分段文件达到一定的条件时进行切分，其对应的索引文件也跟着切分.

日志分段文件切分包含以下几个条件，满足其一即可：

1. 当前日志分段文件的大小超过了 broker 端参数 `log.segment.bytes`配置的值。`log.segment.bytes` 参数的默认值为1073741824，即1GB。
2. 当前日志分段中消息的最大时间戳与当前系统的时间戳的差值大于 `log.roll.ms` 或 `log.roll.hours` 参数配置的值。如果同时配置了 `log.roll.ms` 和 `log.roll.hours` 参数，那么 `log.roll.ms` 的优先级高。默认情况下，只配置了 `log.roll.hours` 参数，其值为168，即7天。
3. 偏移量索引文件或时间戳索引文件的大小达到 broker 端参数 `log.index.size.max.bytes`配置的值(默认值为10485760，即10MB)。
4. 追加的消息的偏移量与当前日志分段的偏移量之间的差值大于 `Integer.MAX_VALUE`，即要追加的消息的偏移量不能转变为相对偏移量（offset - baseOffset > Integer.MAX_VALUE）。



## 3.3 偏移量索引

![image-20210831163642759](img/image-20210831163642759.png)

每个偏移量索引项的格式:

- relativeOffset：相对偏移量，4个字节. 表示消息相对于 baseOffset 的偏移量，当前索引文件的文件名即为 baseOffset 的值。

- position：物理地址，4个字节, 也就是消息在日志分段文件中对应的物理位置。



假设00000000000000000000.index, 截取的内容是16进制表示的:

```
0000 0006 0000 009c 
0000 000e 0000 01cb
0000 0016 0000 02fa 
0000 001a 0000 03b0 
0000 001f 0000 0475
```

使用` kafka-dump-log.sh`解析

```
[root@node1 kafka_2.11-2.0.0]# bin/kafka-dump-log.sh --files /tmp/kafka-logs/ topic-log-0/00000000000000000000.index
Dumping /tmp/kafka-logs/topic-log-0/00000000000000000000.index
offset: 6 position: 156
offset: 14 position: 459
offset: 22 position: 656
offset: 26 position: 838
offset: 31 position: 1050
```



<img src="img/image-20210831164820722.png" alt="image-20210831164820722" style="zoom:70%;" />




## 3.4 时间戳索引

![image-20210831164913000](img/image-20210831164913000.png)

每个索引项占用12个字节，分为两个部分。

1. timestamp：当前日志分段最大的时间戳。
2. relativeOffset：时间戳所对应的消息的相对偏移量。



<img src="img/image-20210831165020021.png" alt="image-20210831165020021" style="zoom:70%;" />





# 4. 日志清理



Kafka 提供了两种日志清理策略：

1. **日志删除**（Log Retention）：按照一定的保留策略直接删除不符合条件的日志分段。
2. **日志压缩**（Log Compaction）：针对每个消息的 key 进行整合，对于有相同 key 的不同 value 值，只保留最后一个版本。



broker 端参数 `log.cleanup.policy` 来设置日志清理策略, 

log.cleanup.policy = **delete**(默认)

log.cleanup.policy = **compact**, 开启压缩,  还需要 log.cleaner.enable=true(默认)



**清理策略可以指定为topic级别.**



## 4.1 日志删除

日志分段的保留策略有3种：基于时间的保留策略、基于日志大小的保留策略和基于日志起始偏移量的保留策略。



### 4.1.1 基于时间

kafka删除保留时间大于设定阈值的日志文件. 

阈值在broker 端参数

- log.retention.hours: 最高优先级, 默认168, 即7天
- log.retention.minutes: 次优先级
- log.retention.ms: 最低优先级

保留时间判断是根据日志分段中最大的时间戳 largestTimeStamp (时间戳索引文件的最后一条索引项的时间)来判断的. 

若所有日志分段都过期, 则根据最后一个分段切分出一个新的日志分段作为activeSegment来接受新消息, 之后再执行删除操作.



**删除操作**

删除日志分段时，首先会从 Log 对象中所维护日志分段的跳跃表中移除待删除的日志分段，以保证没有线程对这些日志分段进行读取操作。然后将日志分段所对应的所有文件添加上“.deleted”的后缀（当然也包括对应的索引文件）。最后交由一个以“delete-file”命名的延迟任务来删除这些以“.deleted”为后缀的文件，这个任务的延迟执行时间可以通过 file.delete.delay.ms 参数来调配，默认值为60000，单位ms.





### 4.1.2 基于日志大小

基于日志大小(`retentionSize `)指的是所有log日志文件的大小, 并不是单个日志分段的大小.

broker 端参数`log.retention.bytes` 来配置日志文件大小，默认值为-1，表示无穷大.

单个日志分段的大小由 broker 端参数 `log.segment.bytes` 来限制，默认值为1073741824，即 1GB.

计算日志文件的总大小 size 和 retentionSize 的差值 diff，即计算需要删除的日志总大小，然后从日志文件中的第一个日志分段开始进行查找可删除的日志分段的文件集合 deletableSegments。





### 4.1.3 **基于日志起始偏移量**



基于日志起始偏移量的保留策略的判断依据是某日志分段的下一个日志分段的起始偏移量 baseOffset 是否小于等于 logStartOffset，若是，则可以删除此日志分段。



## 4.2 日志压缩



针对每个消息的 key 进行整合，对于有相同 key 的不同 value 值，只保留最后一个版本。

适合持久化存储数据, 例如用户的信息等. 







# 5. 磁盘存储



日志切片顺序写, 零拷贝, 



# 6. 时间轮

Kafka 中存在大量的延时操作，比如延时生产、延时拉取和延时删除等。Kafka 的延时功能定时器（SystemTimer）是基于时间轮概念而自定义实现。JDK 中 Timer 和 DelayQueue 的插入和删除操作的平均时间复杂度为 O(nlogn) 并不能满足 Kafka 的高性能要求，而基于时间轮可以将插入和删除操作的时间复杂度都降为 O(1)。



<img src="img/image-20210901160806568.png" alt="image-20210901160806568" style="zoom:80%;" />

Kafka 中的时间轮（TimingWheel）是一个存储定时任务的环形队列，底层采用数组实现，数组中的每个元素可以存放一个定时任务列表（TimerTaskList）。TimerTaskList 是一个环形的双向链表，链表中的每一项表示的都是定时任务项（TimerTaskEntry），其中封装了真正的定时任务（TimerTask）。

时间轮由多个时间格组成，每个时间格代表当前时间轮的基本时间跨度（tickMs）。时间轮的时间格个数是固定的，可用 wheelSize 来表示，那么整个时间轮的总体时间跨度（interval）可以通过公式 tickMs×wheelSize 计算得出。

时间轮还有一个表盘指针（currentTime），用来表示时间轮当前所处的时间，currentTime 是 tickMs 的整数倍。currentTime 可以将整个时间轮划分为到期部分和未到期部分，currentTime 当前指向的时间格也属于到期部分，表示刚好到期，需要处理此时间格所对应的 TimerTaskList 中的所有任务。

若时间轮的 tickMs 为 1ms 且 wheelSize 等于20，那么可以计算得出总体时间跨度 interval 为20ms。

初始情况下表盘指针 currentTime 指向时间格0，此时有一个定时为2ms的任务插进来会存放到时间格为2的 TimerTaskList 中。随着时间的不断推移，指针 currentTime 不断向前推进，过了2ms之后，当到达时间格2时，就需要将时间格2对应的 TimeTaskList 中的任务进行相应的到期操作。此时若又有一个定时为8ms的任务插进来，则会存放到时间格10中，currentTime 再过8ms后会指向时间格10。

如果同时有一个定时为19ms的任务插进来怎么办？新来的 TimerTaskEntry 会复用原来的 TimerTaskList，所以它会插入原本已经到期的时间格1。总之，整个时间轮的总体跨度是不变的，随着指针 currentTime 的不断推进，当前时间轮所能处理的时间段也在不断后移，总体时间范围在 currentTime 和 currentTime+interval 之间。

如果此时有一个定时为 350ms 的任务该如何处理？直接扩充 wheelSize 的大小？Kafka 中不乏几万甚至几十万毫秒的定时任务，这个 wheelSize 的扩充没有底线，就算将所有的定时任务的到期时间都设定一个上限，比如100万毫秒，那么这个 wheelSize 为100万毫秒的时间轮不仅占用很大的内存空间，而且也会拉低效率。Kafka 为此引入了层级时间轮的概念，当任务的到期时间超过了当前时间轮所表示的时间范围时，就会尝试添加到上层时间轮中。

<img src="img/image-20210901161033737.png" alt="image-20210901161033737" style="zoom:80%;" />

如上图所示，复用之前的案例，第一层的时间轮 tickMs=1ms、wheelSize=20、interval=20ms。第二层的时间轮的 tickMs 为第一层时间轮的 interval，即20ms。每一层时间轮的 wheelSize 是固定的，都是20，那么第二层的时间轮的总体时间跨度 interval 为400ms。以此类推，这个400ms也是第三层的 tickMs 的大小，第三层的时间轮的总体时间跨度为8000ms。

对于之前所说的 350ms 的定时任务，显然第一层时间轮不能满足条件，所以就升级到第二层时间轮中，最终被插入第二层时间轮中时间格17所对应的 TimerTaskList。如果此时又有一个定时为 450ms 的任务，那么显然第二层时间轮也无法满足条件，所以又升级到第三层时间轮中，最终被插入第三层时间轮中时间格1的 TimerTaskList。注意到在到期时间为 [400ms,800ms) 区间内的多个任务（比如 446ms、455ms 和 473ms 的定时任务）都会被放入第三层时间轮的时间格1，时间格1对应的 TimerTaskList 的超时时间为 400ms。

随着时间的流逝，当此 TimerTaskList 到期之时，原本定时为 450ms 的任务还剩下 50ms 的时间，还不能执行这个任务的到期操作。这里就有一个时间轮降级的操作，会将这个剩余时间为 50ms 的定时任务重新提交到层级时间轮中，此时第一层时间轮的总体时间跨度不够，而第二层足够，所以该任务被放到第二层时间轮到期时间为 [40ms,60ms) 的时间格中。再经历 40ms 之后，此时这个任务又被“察觉”，不过还剩余 10ms，还是不能立即执行到期操作。所以还要再有一次时间轮的降级，此任务被添加到第一层时间轮到期时间为 [10ms,11ms) 的时间格中，之后再经历 10ms 后，此任务真正到期，最终执行相应的到期操作。



# 7. 延时操作

在 Kafka 中有多种延时操作，延时生产(DelayedProduce)，延时拉取（DelayedFetch）、延时数据删除（DelayedDeleteRecords）等。

延时操作需要延时返回响应的结果，首先它必须有一个超时时间（delayMs），如果在这个超时时间内没有完成既定的任务，那么就需要强制完成以返回响应结果给客户端。其次，延时操作不同于定时操作，定时操作是指在特定时间之后执行的操作，而延时操作可以在所设定的超时时间之前完成，所以延时操作能够支持外部事件的触发。



说下**延时生产**和**延时拉取.** 



## 延时生产

生产者客户端发送消息的时候将 acks 参数设置为-1，那么就意味着需要等待 ISR 集合中的所有副本都确认收到消息之后才能正确地收到响应的结果，或者捕获超时异常。

将消息写入 leader 副本的本地日志文件之后，Kafka 会创建一个延时的生产操作（DelayedProduce），用来处理消息正常写入所有副本或超时的情况，以返回相应的响应结果给客户端。如果在超时时间内始终无法完成，则强制执行。

就延时生产操作而言，它的**外部事件**是所要写入消息的某个分区的 HW（高水位）发生增长。也就是说，随着 follower 副本不断地与 leader 副本进行消息同步，进而促使HW(高水位)进一步增长，HW 每增长一次都会检测是否能够完成此次延时生产操作，如果可以就执行以此返回响应结果给客户端；如果在超时时间内始终无法完成，则强制执行。

<img src="img/image-20210901163153645.png" alt="image-20210901163153645" style="zoom: 80%;" />![image-20210901163407805](img/image-20210901163407805.png)

<img src="img/image-20210901163407805.png" alt="" style="zoom: 80%;" />![image-20210901163407805](img/image-20210901163407805.png)





## 延时拉取

两个 follower 副本都已经拉取到了 leader 副本的最新位置，此时又向 leader 副本发送拉取请求，而 leader 副本并没有新的消息写入，那么此时 leader 副本该如何处理呢？可以直接返回空的拉取结果给 follower 副本，不过在 leader 副本一直没有新消息写入的情况下，follower 副本会一直发送拉取请求，并且总收到空的拉取结果，这样徒耗资源，显然不太合理。

Kafka 选择了延时操作来处理这种情况。Kafka 在处理拉取请求时，会先读取一次日志文件，如果收集不到足够多（fetchMinBytes，由参数 fetch.min.bytes 配置，默认值为1）的消息，那么就会创建一个延时拉取操作（DelayedFetch）以等待拉取到足够数量的消息。当延时拉取操作执行时，会再读取一次日志文件，然后将拉取结果返回给 follower 副本。延时拉取操作也会有一个专门的延时操作管理器负责管理，大体的脉络与延时生产操作相同，不再赘述。如果拉取进度一直没有追赶上leader副本，那么在拉取 leader 副本的消息时一般拉取的消息大小都会不小于 fetchMinBytes，这样 Kafka 也就不会创建相应的延时拉取操作，而是立即返回拉取结果。

延时拉取的外部事件:

- 如果是 follower 副本的延时拉取，它的外部事件就是消息追加到了 leader 副本的本地日志文件中；
- 如果是消费者客户端的延时拉取，它的外部事件可以简单地理解为HW的增长。



# 8. 控制器

在 Kafka 集群中会有一个或多个 broker，其中有一个 broker 会被选举为控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态。

- 当某个分区的 leader 副本出现故障时，由控制器负责为该分区选举新的 leader 副本。
- 当检测到某个分区的 ISR 集合发生变化时，由控制器负责通知所有broker更新其元数据信息。
- 当使用 kafka-topics.sh 脚本为某个 topic 增加分区数量时，同样还是由控制器负责分区的重新分配。



## 1. 控制器的选举及异常恢复

Kafka 中的控制器选举工作依赖于 ZooKeeper，成功竞选为控制器的 broker 会在 ZooKeeper 中创建 /controller 这个临时节点，此临时节点的内容参考如下：

```
{"version":1,"brokerid":0,"timestamp":"1529210278988"}
```

其中 version 在目前版本中固定为1，brokerid 表示成为控制器的 broker 的 id 编号，timestamp 表示竞选成为控制器时的时间戳。

在任意时刻，集群中有且仅有一个控制器。

每个 broker 启动的时候会去尝试读取 /controller 节点的 brokerid 的值，

- 如果读取到 brokerid 的值不为-1，则表示已经有其他 broker 节点成功竞选为控制器，所以当前 broker 就会放弃竞选；
- 如果 ZooKeeper 中不存在 /controller 节点，或者这个节点中的数据异常，那么就会尝试去创建 /controller 节点。
- 当前多个broker 去创建节点的时候，只有创建成功的那个 broker 才会成为控制器，而创建失败的 broker 竞选失败。
- 每个 broker 都会在内存中保存当前控制器的 brokerid 值，这个值可以标识为 activeControllerId。



ZooKeeper 中还有一个与控制器有关的 /controller_epoch 节点，这个节点是持久（PERSISTENT）节点，节点中存放的是一个整型的 controller_epoch 值。controller_epoch 用于记录控制器发生变更的次数，即记录当前的控制器是第几代控制器，我们也可以称之为“控制器的纪元”。

- controller_epoch 的初始值为1，即集群中第一个控制器的纪元为1，

- 当控制器发生变更时，每选出一个新的控制器就将该字段值加1。

每个和控制器交互的请求都会携带 controller_epoch 这个字段，Kafka 通过 controller_epoch 来保证控制器的唯一性，进而保证相关操作的一致性.

- 如果请求的 controller_epoch 值小于内存中的 controller_epoch 值，则认为这个请求是向已经过期的控制器所发送的请求，那么这个请求会被认定为无效的请求。
- 如果请求的 controller_epoch 值大于内存中的 controller_epoch 值，那么说明已经有新的控制器当选了。



## 2. 分区leader选举

分区 leader 副本的选举由控制器负责具体实施. 



### OfflinePartitionLeaderElectionStrategy

当创建分区（创建主题或增加分区都有创建分区的动作）或分区上线（比如分区中原先的 leader 副本下线，此时分区需要选举一个新的 leader 上线来对外提供服务）的时候都需要执行 leader 的选举动作，对应的选举策略为 `OfflinePartitionLeaderElectionStrategy`。这种策略的基本思路是**按照 AR 集合中副本的顺序查找第一个存活的副本，并且这个副本在 ISR 集合中**。



**eg**: 假设集群中有3个节点：broker0、broker1 和 broker2.  在某一时刻具有3个分区且副本因子为3的主题 topic-leader 的具体信息如下：

```shell
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-leader
Topic:topic-leader	PartitionCount:3	ReplicationFactor:3	Configs: 
    Topic: topic-leader	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 2,0,1
    Topic: topic-leader	Partition: 1	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
    Topic: topic-leader	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 0,2,1
```

此时关闭 broker0，那么对于分区2而言，存活的 AR 就变为[1,2]，同时 ISR 变为[2,1]。此时查看主题 topic-leader 的具体信息（参考如下），分区2的 leader 就变为了1而不是2。

```shell
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-leader
Topic:topic-leader	PartitionCount:3	ReplicationFactor:3	Configs: 
    Topic: topic-leader	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 2,1
    Topic: topic-leader	Partition: 1	Leader: 2	Replicas: 2,0,1	Isr: 2,1
    Topic: topic-leader	Partition: 2	Leader: 1	Replicas: 0,1,2	Isr: 2,1
```



### ReassignPartitionLeaderElectionStrategy

当分区进行重分配的时候也需要执行 leader 的选举动作，对应的选举策略为`ReassignPartitionLeaderElectionStrategy`。这个选举策略的思路比较简单：**从重分配的AR列表中找到第一个存活的副本，且这个副本在目前的 ISR 列表中**。

###　ControlledShutdownPartitionLeaderElectionStrategy

还有一种情况会发生 leader 的选举，当某节点被优雅地关闭（也就是执行 ControlledShutdown）时，位于这个节点上的 leader 副本都会下线，所以与此对应的分区需要执行 leader 的选举。与此对应的选举策略（`ControlledShutdownPartitionLeaderElectionStrategy`）为：**从 AR 列表中找到第一个存活的副本，且这个副本在目前的 ISR 列表中，与此同时还要确保这个副本不处于正在被关闭的节点上**。





# 9. 服务器端重要参数

| 参 数 名 称                             | 默 认 值                              | 参 数 释 义                                                  |
| --------------------------------------- | ------------------------------------- | ------------------------------------------------------------ |
| auto.create.topics.enable               | true                                  | 是否开启自动创建主题的功能                                   |
| auto.leader.rebalance.enable            | true                                  | 是否开始自动leader再均衡的功能                               |
| background.threads                      | 10                                    | 指定执行后台任务的线程数                                     |
| compression.type                        | producer                              | 消息的压缩类型。Kafka支持的压缩类型有Gzip、Snappy、LZ4等。默认值“producer”表示根据生产者使用的压缩类型压缩，也就是说，生产者不管是否压缩消息，或者使用何种压缩方式都会被broker端继承。“uncompressed”表示不启用压缩 |
| delete.topic.enable                     | true                                  | 是否可以删除主题                                             |
| leader.imbalance.check.interval.seconds | 300                                   | 检查leader是否分布不均衡的周期                               |
| leader.imbalance.per.broker.percentage  | 10                                    | 允许leader不均衡的比例，若超过这个值就会触发leader再均衡的操作（前提是auto.leader.rebalance.enable参数也要设定为true） |
| log.flush.interval.messages             | 9223372036854775807（Long.MAX_VALUE） | 如果日志文件中的消息在存入磁盘前的数量达到这个参数所设定的阈值时，则会强制将这些刷新日志文件到磁盘中。消息在写入磁盘前还要经历一层操作系统页缓存，如果期间发生掉电，则这些页缓存中的消息会丢失，调小这个参数的大小会增大消息的可靠性，但也会降低系统的整体性能 |
| log.flush.interval.ms                   | null                                  | 刷新日志文件的时间间隔。如果没有配置这个值，则会依据log.flush. scheduler.interval.ms参数设置的值来运作 |
| log.flush.scheduler.interval.ms         | 9223372036854775807（Long.MAX_VALUE） | 检查日志文件是否需要刷新的时间间隔                           |
| log.retention.bytes                     | -1                                    | 日志文件的最大保留大小（分区级别，注意与log.segment.bytes的区别） |
| log.retention.hours 168                 | （7天）                               | 日志文件的留存时间，单位为小时                               |
| log.retention.minutes                   | null                                  | 日志文件的留存时间，单位为分钟                               |
| log.retention.ms                        | null                                  | 日志文件的留存时间，单位为毫秒。log.retention.{hours         |
| log.roll.hours                          | 168（7天）                            | 经过多长时间之后会强制新建一个日志分段，默认值为7            |
| log.roll.ms                             | null                                  | 同上，不过单位为毫秒。优先级比log.roll.hours要高             |
| log.segment.bytes                       | 1073741824（1GB）                     | 日志分段文件的最大值，超过这个值会强制创建一个新的日志分段   |
| log.segment.delete.delay.ms             | 60000（60秒）                         | 从操作系统删除文件前的等待时间                               |
| min.insync.replicas                     | 1                                     | ISR集合中最少的副本数                                        |
| num.io.threads                          | 8                                     | 处理请求的线程数，包含磁盘I/O                                |
| num.network.threads                     | 3                                     | 处理接收和返回响应的线程数                                   |
| log.cleaner.enable                      | true                                  | 是否开启日志清理的功能                                       |
| log.cleaner.min.cleanable.ratio         | 0.5                                   | 限定可执行清理操作的最小污浊率                               |
| log.cleaner.threads                     | 1                                     | 用于日志清理的后台线程数                                     |
| log.cleanup.policy                      | delete                                | 日志清理策略，还有一个可选项为compact，表示日志压缩          |
| log.index.interval.bytes                | 4096                                  | 每隔多少个字节的消息量写入就添加一条索引                     |
| log.index.size.max.bytes                | 10485760（10MB）                      | 索引文件的最大值                                             |
| log.message.format.version              | 2.0-IV1                               | 消息格式的版本                                               |
| log.message.timestamp.type              | CreateTime                            | 消息中的时间戳类型，另一个可选项为LogAppendTime。CreateTime表示消息创建的时间，LogAppendTime表示消息追加到日志中的时间 |
| log.retention.check.interval.ms         | 300000（5分钟）                       | 日志清理的检查周期                                           |
| num.partitions                          | 1                                     | 主题中默认的分区数                                           |
| reserved.broker.max.id                  | 1000                                  | broker.id能配置的最大值，同时reserved.broker.max.id+1也是自动创建broker.id值的起始大小，详细参考6.5.1节 |
| create.topic.policy.class.name          | null                                  | 创建主题时用来验证合法性的策略，这个参数配置的是一个类的全限定名，需要实现org.apache.kafka.server. policy.CreateTopicPolicy接口 |
| broker.id.generation.enable             | true                                  | 是否开启自动生成broker.id的功能                              |
| broker.rack                             | null                                  | 配置broker的机架信息                                         |



# 10. 消费端分区分配策略

消费者客户端参数 `partition.assignment.strategy` 来设置消费者与订阅主题之间的分区分配策略。默认`RangeAssignor` 策略。

除此之外，Kafka 还提供了另外两种分配策略：`RoundRobinAssignor` 和 `StickyAssignor`。

消费者客户端参数 `partition.assignment.strategy` 可以配置多个分配策略，彼此之间以逗号分隔。



## 1. RangeAssignor配策略

`RangeAssignor`策略的原理是**按照消费者总数和分区总数进行整除运算来获得一个跨度，然后将分区按照跨度进行平均分配**。

对于每一个主题，`RangeAssignor `策略会将消费组内所有订阅这个主题的消费者按照名称的字典序排序，然后为每个消费者划分固定的分区范围，如果不够平均分配，那么字典序靠前的消费者会被多分配一个分区。 此策略有可能导致部分消费者过载的情况.

eg: 

假设消费组内有2个消费者 C0 和 C1，都订阅了主题 t0 和 t1，并且每个主题都有4个分区，那么订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t0p3、t1p0、t1p1、t1p2、t1p3。最终的分配结果为：

```
消费者C0：t0p0、t0p1、t1p0、t1p1
消费者C1：t0p2、t0p3、t1p2、t1p3
```

这样分配得很均匀，那么这个分配策略能够一直保持这种良好的特性吗？我们不妨再来看另一种情况。假设上面例子中2个主题都只有3个分区，那么订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终的分配结果为：

```
消费者C0：t0p0、t0p1、t1p0、t1p1
消费者C1：t0p2、t1p2
```





## 2.RoundRobinAssignor策略

`RoundRobinAssignor` 分配策略的原理是**将消费组内所有消费者及消费者订阅的所有主题的分区按照字典序排序，然后通过轮询方式逐个将分区依次分配给每个消费者**。

如果同一个消费组内所有的消费者的订阅信息都是相同的，`那么 RoundRobinAssignor` 分配策略的分区分配会是均匀的。

假设消费组中有2个消费者 C0 和 C1，都订阅了主题 t0 和 t1，并且每个主题都有3个分区，那么订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终的分配结果为：

```
消费者C0：t0p0、t0p2、t1p1
消费者C1：t0p1、t1p0、t1p2
```



如果同一个消费组内的消费者订阅的信息是不相同的，那么在执行分区分配的时候就不是完全的轮询分配，有可能导致分区分配得不均匀。

假设消费组内有3个消费者（C0、C1 和 C2），它们共订阅了3个主题（t0、t1、t2），这3个主题分别有1、2、3个分区，即整个消费组订阅了 t0p0、t1p0、t1p1、t2p0、t2p1、t2p2 这6个分区。具体而言，消费者 C0 订阅的是主题 t0，消费者 C1 订阅的是主题 t0 和 t1，消费者 C2 订阅的是主题 t0、t1 和 t2，那么最终的分配结果为：

```
消费者C0：t0p0
消费者C1：t1p0
消费者C2：t1p1、t2p0、t2p1、t2p2
```





## 3. StickyAssignor策略 (粘性策略)

`StickyAssignor `分配策略主要有两个目的：

1. 分区的分配要尽可能均匀。
2. 分区的分配尽可能与上次分配的保持相同。

当两者发生冲突时，第一个目标优先于第二个目标。

如果发生分区重分配，StickyAssignor 分配策略如同其名称中的“sticky”一样，让分配策略具备一定的“黏性”，尽可能地让同一消费者的前后两次分配相同，进而减少系统资源的损耗及其他异常情况的发生。

**StickyAssignor 分配策略比另外两者分配策略而言显得更加优异**, 有兴趣可以研究一下策略的实现代码, 会比前两个策略复杂不少.





# 11. 消费者协调器和组协调器

新版本kafka(2.0+)将全部消费组分成多个子集，每个消费组的子集在服务端对应一个 GroupCoordinator 对其进行管理，GroupCoordinator 是 Kafka 服务端中用于管理消费组的组件。而消费者客户端中的 ConsumerCoordinator 组件负责与 GroupCoordinator 进行交互。

ConsumerCoordinator 与 GroupCoordinator 之间最重要的职责就是**负责执行消费者再均衡(rebalance)的操作**，包括前面提及的分区分配的工作也是在再均衡期间完成的。就目前而言，一共有如下几种情形会触发再均衡的操作：

- 有新的消费者加入消费组。
- 有消费者宕机下线。消费者并不一定需要真正下线，例如遇到长时间的GC、网络延迟导致消费者长时间未向 GroupCoordinator 发送心跳等情况时，GroupCoordinator 会认为消费者已经下线。
- 有消费者主动退出消费组（发送 LeaveGroupRequest 请求）。比如客户端调用了 unsubscrible() 方法取消对某些主题的订阅。
- 消费组所对应的 GroupCoorinator 节点发生了变更。
- 消费组内所订阅的任一主题或者主题的分区数量发生变化。



当有消费者加入消费组时，消费者、消费组及组协调器之间的**再均衡操作**会经历一下几个阶段。

##　第一阶段（FIND_COORDINATOR）

消费者需要确定它所属的消费组对应的 GroupCoordinator 所在的 broker，并创建与该 broker 相互通信的网络连接。如果消费者已经保存了与消费组对应的 GroupCoordinator 节点的信息，并且与它之间的网络连接是正常的，那么就可以进入第二阶段。否则，就需要向集群中的某个节点发送 FindCoordinatorRequest 请求来查找对应的 GroupCoordinator，这里的“某个节点”并非是集群中的任意节点，而是负载最小的节点。



Kafka 在收到 FindCoordinatorRequest 请求之后，会根据 coordinator_key（也就是 groupId）查找对应的 GroupCoordinator 节点，如果找到对应的 GroupCoordinator 则会返回其相对应的 node_id、host 和 port信息。



##　第二阶段（JOIN_GROUP）

在成功找到消费组所对应的 GroupCoordinator 之后就进入加入消费组的阶段，在此阶段的消费者会向 GroupCoordinator 发送 JoinGroupRequest 请求，并处理响应。

如果是原有的消费者重新加入消费组，那么在真正发送 JoinGroupRequest 请求之前还要执行一些准备工作：

1. 如果消费端参数 enable.auto.commit 设置为 true（默认值也为 true），即开启自动提交位移功能，那么在请求加入消费组之前需要向 GroupCoordinator 提交消费位移。这个过程是阻塞执行的，要么成功提交消费位移，要么超时。
2. 如果消费者添加了自定义的再均衡监听器（ConsumerRebalanceListener），那么此时会调用 onPartitionsRevoked() 方法在重新加入消费组之前实施自定义的规则逻辑，比如清除一些状态，或者提交消费位移等。
3. 因为是重新加入消费组，之前与 GroupCoordinator 节点之间的心跳检测也就不需要了，所以在成功地重新加入消费组之前需要禁止心跳检测的运作。

消费者在发送 JoinGroupRequest 请求之后会阻塞等待 Kafka 服务端的响应。服务端在收到 JoinGroupRequest 请求后会交由 GroupCoordinator 来进行处理。

**选举消费组的leader**

GroupCoordinator 需要为消费组内的消费者选举出一个消费组的 leader．

**选举分区分配策略**

每个消费者都可以设置自己的分区分配策略，对消费组而言需要从各个消费者呈报上来的各个分配策略中选举一个彼此都“信服”的策略来进行整体上的分区分配。这个分区分配的选举并非由 leader 消费者决定，而是根据消费组内的各个消费者投票来决定的。这里所说的“根据组内的各个消费者投票来决定”不是指 GroupCoordinator 还要再与各个消费者进行进一步交互，而是根据各个消费者呈报的分配策略来实施。最终选举的分配策略基本上可以看作被各个消费者支持的最多的策略，具体的选举过程如下：

1. 收集各个消费者支持的所有分配策略，组成候选集 candidates。
2. 每个消费者从候选集 candidates 中找出第一个自身支持的策略，为这个策略投上一票。
3. 计算候选集中各个策略的选票数，选票数最多的策略即为当前消费组的分配策略。





##　第三阶段（SYNC_GROUP）

各个消费者会向 GroupCoordinator 发送 SyncGroupRequest 请求来同步分配方案．

leader 消费者根据在第二阶段中选举出来的分区分配策略来实施具体的分区分配，在此之后需要将分配的方案同步给各个消费者，此时 leader 消费者并不是直接和其余的普通消费者同步分配方案，而是通过 GroupCoordinator 这个“中间人”来负责转发同步分配方案的。





##　第四阶段（HEARTBEAT）

进入这个阶段之后，消费组中的所有消费者就会处于正常工作状态。

消费者通过向 GroupCoordinator 发送心跳来维持它们与消费组的从属关系，以及它们对分区的所有权关系。只要消费者以正常的时间间隔发送心跳，就被认为是活跃的，说明它还在读取分区中的消息。**心跳线程是一个独立的线程**，可以在轮询消息的空档发送心跳。如果消费者停止发送心跳的时间足够长，则整个会话就被判定为过期，GroupCoordinator 也会认为这个消费者已经死亡，就会触发一次再均衡行为。

消费者的心跳间隔时间由参数 `heartbeat.interval.ms` 指定，默认3秒，这个参数必须比 `session.timeout.ms` 参数设定的值要小(`一般情况下 heartbeat.interval.ms 的配置值不能超过 session.timeout.ms 配置值的1/3。这个参数可以调整得更低，以控制正常重新平衡的预期时间。`)

如果一个消费者发生崩溃，并停止读取消息，那么 GroupCoordinator 会等待一小段时间，确认这个消费者死亡之后才会触发再均衡。在这一小段时间内，死掉的消费者并不会读取分区里的消息。这个一小段时间由 `session.timeout.ms `参数控制，该参数的配置值必须在broker端参数 `group.min.session.timeout.ms`（默认6秒）和 `group.max.session.timeout.ms`（默认5分钟）允许的范围内。

还有一个参数 `max.poll.interval.ms`，它用来指定使用消费者组管理时 **poll() 方法调用之间的最大延迟**，也就是消费者在获取更多消息之前可以空闲的时间量的上限。如果此超时时间期满之前 poll() 没有调用，则消费者被视为失败，并且分组将重新平衡，以便将分区重新分配给别的成员。





# 12. __consumer_offsets队列

当集群中第一次有消费者消费消息时会自动创建主题 __consumer_offsets，不过它的副本因子通过 offsets.topic.replication.factor 参数设置(默认为3)，分区数可以通过 offsets.topic.num.partitions 参数设置(默认为50)。



我们可以通过 kafka-console-consumer.sh 脚本来查看 __consumer_offsets 中的内容，不过要设定 formatter 参数为 kafka.coordinator.group.GroupMetadataManager$OffsetsMessageFormatter。假设我们要查看消费组“consumerGroupId”的位移提交信息，首先可以根据代码清单12-1中的计算方式得出分区编号为20，然后查看这个分区中的消息，

```shell
[root@node1 kafka_2.11-2.0.0]# bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic __consumer_offsets -–partition 20 --formatter 'kafka.coordinator.group.GroupMetadataManager$OffsetsMessageFormatter'

[consumerGroupId,topic-offsets,30]::[OffsetMetadata[2130,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,8]::[OffsetMetadata[2310,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,21]::[OffsetMetadata[1230,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,27]::[OffsetMetadata[1230,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,9]::[OffsetMetadata[1233,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,35]::[OffsetMetadata[1230,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,41]::[OffsetMetadata[3210,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,33]::[OffsetMetadata[1310,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
[consumerGroupId,topic-offsets,23]::[OffsetMetadata[2123,NO_METADATA],CommitTime 1538843128354,ExpirationTime 1539447928354]
（…省略若干）
```



消费者组在`__consumer_offsets`的分区号, 可以通过下面方法计算.

```
Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount
```

其中 groupId.hashCode 就是使用 Java 中 String 类的 hashCode() 方法获得的，groupMetadataTopicPartitionCount 为主题 `__consumer_offsets` 的分区个数(默认值为50)。



OffsetsMessageFormatter 打印的日志格式：

```shell
"[%s,%s,%d]:: [OffsetMetadata[%d,%s],CommitTime %d,ExpirationTime %d]".format (group, topic, partition, offset, metadata, commitTimestamp, expireTimestamp)
```



# 13. 事务



## 消息传输保障

一般而言，消息中间件的消息传输保障有3个层级，分别如下。

- at most once：至多一次。消息可能会丢失，但绝对不会重复传输。

- at least once：最少一次。消息绝不会丢失，但可能会重复传输。

- exactly once：恰好一次。每条消息肯定会被传输一次且仅传输一次。

为了实现`exactly once, 恰好一次`, Kafka 引入了幂等和事务这两个特性。



## 幂等

开启幂等, 需要显式地将生产者客户端参数 `enable.idempotence` 设置为 true 即可（这个参数的默认值为 false）.

开启幂等后, 会默认配置生产者客户端`retries`、`acks`、`max.in.flight.requests.per.connection`这三个参数. 如果有手动配置, 需要检查不被配置错. 

- retries 参数的值必须大于0, 开启幂等会配置为 `Integer.MAX_VALUE`.
- acks 参数必须是-1. 
- max.in.flight.requests.per.connection 不能大于5（值默认为5). 该参数指定了**生产者在收到服务器响应之前可以发送多少个消息**。它的值越高，就会占用越多的内存，不过也会提升吞吐量.



为了实现生产者的幂等性，Kafka 为此引入了 `producer id`（以下简称 PID）和序列号（`sequence number`）这两个概念.

每个新的生产者实例在初始化的时候都会被分配一个 PID，这个 PID 对用户而言是完全透明的。对于**每个 PID，消息发送到的每一个分区都有对应的序列号，这些序列号从0开始单调递增**。生产者每发送一条消息就会将 <PID，分区> 对应的序列号的值加1。

broker 端会在内存中为每一对 <PID，分区> 维护一个序列号。对于收到的每一条消息，

- 只有当它的序列号的值（SN_new）比 broker 端中维护的对应的序列号的值（SN_old）大1（即 SN_new = SN_old + 1）时，broker 才会接收它。
- 如果 SN_new< SN_old + 1，那么说明消息被重复写入，broker 可以直接将其丢弃。
- 如果 SN_new> SN_old + 1，那么说明中间有数据尚未写入，出现了乱序，暗示可能有消息丢失，对应的生产者会抛出 `OutOfOrderSequenceException`.



**注意**

引入序列号来实现幂等也只是针对每一对 <PID，分区> 而言的，也就是说，**Kafka 的幂等只能保证单个生产者会话（session）中单分区的幂等**。

多次send相同内容的消息, 对kafka来说是两条消息, 因此分配给消息的序列号不一样.



## 事务

Kafka中的事务可以使应用程序将消费消息、生产消息、提交消费位移当作原子操作来处理，同时成功或失败，即使该生产或消费会跨多个分区。

为了实现事务，应用程序必须提供唯一的 transactionalId，这个 transactionalId 通过客户端参数 `transactional.id` 来显式设置.

```java
properties.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "transactionId");
```

事务要求生产者开启幂等特性，因此设置了`transactional.id`, 若没设置`enable.idempotence`为 true, kafkaProducer会默认设置为true.



从消费者的角度分析，事务能保证的语义相对偏弱。出于以下原因，Kafka 并不能保证已提交的事务中的所有消息都能够被消费：

- 对采用日志压缩策略的主题而言，事务中的某些消息有可能被清理（相同key的消息，后写入的消息会覆盖前面写入的消息）。
- 事务中消息可能分布在同一个分区的多个日志分段（LogSegment）中，当老的日志分段被删除时，对应的消息可能会丢失。
- 消费者可以通过 seek() 方法访问任意 offset 的消息，从而可能遗漏事务中的部分消息。
- 消费者在消费时可能没有分配到事务内的所有分区，如此它也就不能读取事务中的所有消息。



**isolation.level**

在消费端有一个参数 `isolation.level`. 对事务消息的消费非常重要

- 值为`read_uncommitted`，意思是说消费端应用可以看到（消费到）未提交的事务，当然对于已提交的事务也是可见的。
- 值为`read_committed`，表示消费端应用不可以看到尚未提交的事务内的消息。

举个例子，如果生产者开启事务并向某个分区值发送3条消息 msg1、msg2 和 msg3，在执行 commitTransaction() 或 abortTransaction() 方法前，设置为“read_committed”的消费端应用是消费不到这些消息的，不过在 **KafkaConsumer 内部会缓存**这些消息，直到生产者执行 commitTransaction() 方法之后它才能将这些消息推送给消费端应用。反之，如果生产者执行了 abortTransaction() 方法，那么 KafkaConsumer 会将这些缓存的消息丢弃而不推送给消费端应用。



# 14. 副本

Kafka 从 0.8 版本开始为分区引入了多副本机制，**提升数据容灾能力**以及实现**故障自动转移**，在 Kafka 集群中某个 broker 节点失效的情况下仍然保证服务可用。

相关概念：

- 副本是相对于分区而言的，即副本是特定分区的副本。
- 一个分区中包含一个或多个副本，其中**一个为 leader 副本，其余为 follower 副本**，各个副本位于不同的 broker 节点中。**只有 leader 副本对外提供服务，follower 副本只负责数据同步**。
- 分区中的所有副本统称为 **AR**(`Assigned Repllicas`)，而 **ISR**(`In-Sync Replicas`) 是指与 leader 副本保持同步状态的副本集合，当然 leader 副本本身也是这个集合中的一员。
- **LEO**(`Log End Offset`) 标识每个分区中最后一条消息的下一个位置，分区的每个副本都有自己的 LEO，ISR 中最小的 LEO 即为 **HW**(`High Watermak`)，俗称高水位，消费者只能拉取到 HW 之前的消息。



## 失效副本

正常情况下，分区的所有副本都处于 ISR 集合中，但是难免会有异常情况发生，从而某些副本被剥离出 ISR 集合中。在 ISR 集合之外，也就是处于同步失效或功能失效（比如副本处于非存活状态）的副本统称为失效副本，**失效副本对应的分区也就称为同步失效分区**，即 `under-replicated` 分区。

Kafka 源码注释中说明了一般有两种情况会导致副本失效：

- **follower 副本进程卡住**，在一段时间内根本没有向 leader 副本发起同步请求，比如频繁的 Full GC。
- follower 副本进程同步过慢，在一段时间内都无法追赶上 leader 副本，比如 I/O 开销过大。



## ISR的伸缩

Kafka 在启动的时候会开启两个与ISR相关的定时任务，名称分别为“`isr-expiration`”和“`isr-change-propagation`”。

`isr-expiration `任务会周期性地检测每个分区是否需要缩减其 ISR 集合。这个周期和 `replica.lag.time.max.ms` 参数有关，大小是这个参数值的一半，默认值为 5000ms。当检测到 ISR 集合中有失效副本时，就会收缩 ISR 集合。

随着 follower 副本不断与 leader 副本进行消息同步，follower 副本的 LEO 也会逐渐后移，并最终追赶上 leader 副本，此时该 follower 副本就有资格进入 ISR 集合。追赶上 leader 副本的判定准则是此副本的 LEO 是否不小于 leader 副本的 **HW**，注意这里并不是和 leader 副本的 LEO 相比。



除此之外，当 ISR 集合发生变更时还会将变更后的记录缓存到 isrChangeSet 中，`isr-change-propagation` 任务会周期性（固定值为 2500ms）地检查 isrChangeSet，如果发现 isrChangeSet 中有 ISR 集合的变更记录，那么它会在 ZooKeeper 的 /isr_change_notification 路径下创建一个以 isr_change_ 开头的持久顺序节点（比如 /isr_change_notification/isr_change_0000000000），并将 isrChangeSet 中的信息保存到这个节点中。

Kafka 控制器为 /isr_change_notification 添加了一个 Watcher，当这个节点中有子节点发生变化时会触发 Watcher 的动作，以此通知控制器更新相关元数据信息并向它管理的 broker 节点发送更新元数据的请求，最后删除 /isr_change_notification 路径下已经处理过的节点。



## 消息追加



![image-20210923195421713](img/image-20210923195421713.png)

某个分区有3个副本分别位于 broker0、broker1 和 broker2 节点中，其中带阴影的方框表示本地副本。假设 broker0 上的副本1为当前分区的 leader 副本，那么副本2和副本3就是 follower 副本，整个消息追加的过程可以概括如下：

1. 生产者客户端发送消息至 leader 副本（副本1）中。
2. 消息被追加到 leader 副本的本地日志，并且会更新日志的偏移量。
3. follower 副本（副本2和副本3）向 leader 副本请求同步数据。
4. leader 副本所在的服务器读取本地日志，并更新对应拉取的 follower 副本的信息。
5. leader 副本所在的服务器将拉取结果返回给 follower 副本。
6. follower 副本收到 leader 副本返回的拉取结果，将消息追加到本地日志中，并更新日志的偏移量信息。



Kafka 的根目录下有 cleaner-offset-checkpoint、log-start-offset-checkpoint、recovery-point-offset-checkpoint 和 replication-offset-checkpoint 四个检查点文件。

`recovery-point-offset-checkpoint` 和 `replication-offset-checkpoint` 这两个文件分别对应了 **LEO** 和 **HW**。Kafka 中会有一个定时任务负责将所有分区的 LEO 刷写到恢复点文件 `recovery-point-offset-checkpoint` 中，定时周期由broker端参数 `log.flush.offset.checkpoint.interval.ms` 来配置，默认值为60000。

还有一个定时任务负责将所有分区的 HW 刷写到复制点文件 `replication-offset-checkpoint` 中，定时周期由 broker 端参数 `replica.high.watermark.checkpoint.interval.ms` 来配置(默认值为5000)。

`log-start-offset-checkpoint` 文件对应 `logStartOffset`（注意不能缩写为 LSO，因为在 Kafka 中 LSO 是 LastStableOffset 的缩写），用来标识日志的起始偏移量。各个副本在变动 LEO 和 HW 的过程中，`logStartOffset` 也有可能随之而动。Kafka也有一个定时任务来负责将所有分区的 `logStartOffset` 书写到起始点文件 `log-start-offset-checkpoint` 中，定时周期由 broker 端参数 `log.flush.start.offset.checkpoint.interval.ms` 来配置(默认值为60000)。



## Leader Epoch的作用



在 0.11.0.0 版本之前，Kafka 使用的是基于 HW 的同步机制。在leader 副本发生切换时候， 有可能出现数据丢失或 leader 副本和 follower 副本数据不一致的问题。

从 0.11.0.0 开始引入了 **leader epoch** 的概念，在**需要截断数据的时候使用 leader epoch 作为参考依据**而不是原本的 HW。leader epoch 代表 leader 的纪元信息（epoch），初始值为0。每当 leader 变更一次，leader epoch 的值就会加1，相当于为 leader 增设了一个版本号。



## 不支持读写分离



主写从读有2个很明显的缺点：

1. 数据一致性问题。数据从主节点转到从节点必然会有一个延时的时间窗口，这个时间窗口会导致主从节点之间的数据不一致。某一时刻，在主节点和从节点中 A 数据的值都为 X，之后将主节点中 A 的值修改为 Y，那么在这个变更通知到从节点之前，应用读取从节点中的 A 数据的值并不为最新的 Y，由此便产生了数据不一致的问题。
2. 延时问题。类似 Redis 这种组件，数据从写入主节点到同步至从节点中的过程需要经历网络→主节点内存→网络→从节点内存这几个阶段，整个过程会耗费一定的时间。而在 Kafka 中，主从同步会比 Redis 更加耗时，它需要经历网络→主节点内存→主节点磁盘→网络→从节点内存→从节点磁盘这几个阶段。对延时敏感的应用而言，主写从读的功能并不太适用。



总的来说，Kafka 只支持主写主读有几个优点：

- 可以简化代码的实现逻辑，减少出错的可能；
- 将负载粒度细化均摊，与主写从读相比，不仅负载效能更好，而且对用户可控；
- 没有延时的影响；
- 在副本稳定的情况下，不会出现数据不一致的情况。



# 15. 可靠性分析



##　日志同步机制

> 日志同步机制的一个基本原则就是：如果告知客户端已经成功提交了某条消息，那么即使 leader 宕机，也要保证新选举出来的 leader 中能够包含这条消息。

follower节点会定期从leader节点同步日志. 

在 Kafka 中动态维护着一个 `ISR` 集合，处于 `ISR` 集合内的节点保持与 `leader` 相同的高水位（HW），**只有位列其中的副本（unclean.leader.election.enable 配置为 false）才有资格被选为新的 leader**。**写入消息时只有等到所有ISR集合中的副本都确认收到之后才能被认为已经提交**。位于 ISR 中的任何副本节点都有资格成为 leader，选举过程简单、开销低，

Kafka 允许宕机副本重新加入 ISR 集合，但在进入 ISR 之前必须保证自己能够重新同步完 leader 中的所有数据。



## 消息确认机制

**生产者客户端参数 acks**

- acks = 1，生产者将消息发送到 leader 副本后不等响应. 无法得知成功/失败.
- acks = 1，生产者将消息发送到 leader 副本，leader 副本在成功写入本地日志之后返回producer成功提交.
- ack = -1，生产者将消息发送到 leader 副本，leader 副本在成功写入本地日志之后还要等待 ISR 中的 follower 副本全部同步完成才返回producer成功提交，即使此时 leader 副本宕机，消息也不会丢失.



##　重试机制

有些发送异常属于可重试异常，比如瞬时的网络故障，一般通过重试就可以解决。

`retries` 参数设置为0，即不进行重试. 

`retry.backoff.ms` 参数用来设定两次重试之间的时间间隔.

KafkaAdminClient 中 retries 参数的默认值为5。





# 16. 实现过期消息

通过消费者拦截器来实现.

消费者拦截器用法的时候就使用了消息 TTL（Time To Live，过期时间），其中通过消息的 timestamp 字段和 ConsumerInterceptor 接口的 onConsume() 方法来实现消息的 TTL 功能。



1. ProducerRecord添加RecordHeaders来保存过期时间

```java
ProducerRecord<String, String> record1 =
        new ProducerRecord<>(topic, 0, System.currentTimeMillis(),
                null, "msg_ttl_1",new RecordHeaders().add(new RecordHeader("ttl", BytesUtils.longToBytes(20))));
```



2. 自定义`TTLConsumerInterceptor`实现`ConsumerInterceptor`拦截器

```java
public class TTLConsumerInterceptor implements ConsumerInterceptor<String, String> {


    @Override
    public ConsumerRecords onConsume(ConsumerRecords<String, String> records) {

        long now = System.currentTimeMillis();
        Map<TopicPartition, List<ConsumerRecord<String, String>>> newRecords = new HashMap<>();

        for (TopicPartition tp : records.partitions()) {
            List<ConsumerRecord<String, String>> tpRecords = records.records(tp);
            List<ConsumerRecord<String, String>> newTpRecords = new ArrayList<>();
            for (ConsumerRecord<String, String> record : tpRecords) {
                Headers headers = record.headers();
                long ttl = -1;
                //判断headers中是否有key为“ttl”的Header
                for (Header header : headers) {
                    if (header.key().equalsIgnoreCase("ttl")) {
                        ttl = BytesUtils.bytesToLong(header.value());
                    }
                }
                //消息超时判定
                if (ttl > 0 && now - record.timestamp() < ttl * 1000) {
                    newTpRecords.add(record);
                } else {//没有设置TTL，不需要超时判定
                    newTpRecords.add(record);
                }
            }
            if (!newTpRecords.isEmpty()) {
                newRecords.put(tp, newTpRecords);
            }
        }
        return new ConsumerRecords<>(newRecords);
    }
 }
```



3. consumer的properties注册拦截器

```java
List interceptors = new ArrayList();
interceptors.add(“com.xxx.kafka.consumer.interceptor.TTLConsumerInterceptor”);
props.put(“interceptor.classes”, interceptors);
```





# 17. 实现延时队列



对kafka producer进行二次封装, 在发送延时消息的时候并不是先投递到要发送的真实主题（real_topic）中，而是先投递到一些 Kafka 内部的主题（delay_topic）中，这些内部主题对用户不可见，然后通过一个自定义的服务拉取这些内部主题中的消息，并将满足条件的消息再投递到要发送的真实的主题中，消费者所订阅的还是真实的主题。



<img src="img/image-20210930172558317.png" alt="image-20210930172558317" style="zoom:50%;" />



<img src="img/image-20210930172627829.png" alt="image-20210930172627829" style="zoom:50%;" />