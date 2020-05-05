# kafka\(2.3.0\)测试报告

## 一. 单服务器\(1.22\)硬件信息

### 1.1 CPU, 内存, 网卡

```text
IP: 192.168.1.22
CPU: Intel(R) Xeon(R) CPU E5-2609 v3 @ 1.90GHz, 12核
内存: 48G 2133 MHz
网卡: I350 Gigabit 千兆网卡, 1000Mb/s = 125MB/s, 网络传输瓶颈是125MB/s
```

### 1.2 磁盘写瓶颈

```text
[root]# dd if=/dev/zero of=test.dd bs=1M count=20000

20000+0 records in
20000+0 records out
20971520000 bytes (21 GB) copied, 104.176 s, 201 MB/s
```

### 1.3 磁盘读瓶颈

```text
[root]# hdparm -tT --direct /dev/mapper/vg_devenv22-lv_home

/dev/mapper/vg_devenv22-lv_home:
Timing O_DIRECT cached reads: 1262 MB in 2.01 seconds = 627.28 MB/sec
Timing O_DIRECT disk reads: 390 MB in 3.00 seconds = 129.87 MB/sec
```

## 二. 单机服务器\(1.22\) – 异步提交模式

### 2.1 生产者客户端数据

#### 2.1.1 参数与部署

生产者关键参数:

```text
acks=1
batch.size=1024*128
linger.ms=10
max.block.ms=10000
```

生产者部署: 分别部署三台物理机器\(1.20, 1.21, 1.23\), coreSize=16的线程池分80000次发送一批1000条\*100Byte的数据.

三台物理机器配置跟1.22服务器一致, 并且内网传输速度实测超过100MB/s

#### 2.1.2 生产者端

服务器1.20, 1.21, 1.23, 提交8kw条100Byte的消息, 耗时如下

\(因为三台机器配置一样, 观察日志耗时差不多, 因此只贴其中一台机器的截图\)

![](../../../.gitbook/assets/4039130-c72299d2cb4a24b7.png)

#### 2.1.3 出现的异常现象\(百兆带宽情况下发生\)

消息是100Byte, max.block.ms=3000的时候, 其中一台生产者程序会出现少量如下异常.

```text
org.apache.kafka.common.errors.TimeoutException: Failed to allocate memory within the configured max blocking time 3000 ms.
```

关于max.block.ms的作用:

```text
The configuration controls how long <code>KafkaProducer.send()</code> and <code>KafkaProducer.partitionsFor(topic)</code> will block. 
These methods can be blocked either because the buffer is full or metadata unavailable. 
Blocking in the user-supplied serializers or partitioner will not be counted against this timeout.
```

对应到我们实际的情况. 虽然测试的是异步模式, 但kafka生产者的IO线程是单线程阻塞模型，若acks&gt;0, 请求等待确认\(若有回调函数, 还会执行回调\)的阻塞会直接导致IO线程阻塞，于是生产者buffer的数据无法被发送. kafka生产者还在不断的被应用调用，因此buffer一直累积并增大，当buffer满的时候，生产者线程会被阻塞，最大阻塞时间为max.block.time，如果时间到达之后还是无法将数据塞入缓冲区，则会抛出一个异常，打印出异常栈.

因此原因很可能是发送的速度跟不上请求消息的速度,使得producer内部buffer满了, 而产生阻塞, 在3s内还处理不过来则抛异常.

**等条件排除法:**

* 在1,2台producer的时候不会报错.
* 3个producer, max.block.ms=3000, acks=0, 不适用回调函数, 异常明显从几百条降为20条.
* 3个producer, max.block.time改为10000ms, 没有报此异常.

**优化目的**: 提高broker响应请求速度

**处理方案**:

* broker服务端

  1.搭建集群, 多分区多broker分担请求流量, 加快请求响应速度

* 客户端

  1.客户端机器尽量不要有其他大负载的程序

  2.消息长度尽量精简, 能减少传输时间, 提高效率

  3.尽量使用异步发送模式, producer参数acks设为0\(无需服务端确认\), 有助于减少请求失败数量.

  4.max.block.ms设根据实际设长一些

### 2.2 kafka服务端数据

#### 2.2.1 CPU 内存

1个producer全力提交数据时候的CPU和内存占用情况 从CPU占用从2%不到的使用率升到50%~70% ![](../../../.gitbook/assets/4039130-8a11b5a1a4bc6cc0.png)

2个producer全力提交数据时候的CPU和内存占用情况 ![](../../../.gitbook/assets/4039130-1949e45fc4a55361.png)

3个producer全力提交数据时候的CPU和内存占用情况 ![](../../../.gitbook/assets/4039130-67e6508e706a1138.png)

#### 2.2.2 网卡速率

1个producer全力输出 ![1&#x4E2A;producer&#x5168;&#x529B;&#x8F93;&#x51FA;](../../../.gitbook/assets/4039130-4ed062c41035dda6.png)

2个producer全力输出 ![2&#x4E2A;producer&#x5168;&#x529B;&#x8F93;&#x51FA;](../../../.gitbook/assets/4039130-7dae6644b11f9abc.png)

3个producer全力输出 ![3&#x4E2A;producer&#x5168;&#x529B;&#x8F93;&#x51FA;](../../../.gitbook/assets/4039130-9bdb47105bb3bf83.png)

6个producer全力输出 ![](../../../.gitbook/assets/4039130-d698b2d695fe2c97.png)

#### 2.2.3 kafka-manager

1个producer全力输出 ![](../../../.gitbook/assets/4039130-86270b37084984ce.png)

2个producer全力输出 ![](../../../.gitbook/assets/4039130-ed24faadef63c63c.png)

3个producer全力输出 ![](../../../.gitbook/assets/4039130-7b4eb3fa832fc439.png)

6个producer全力输出 ![](../../../.gitbook/assets/4039130-d388c12d217abc06.png)

### 2.3 异步提交模式-kafka单服务器的结论

经过三四次测试, 得出以下结论关于kafka单服务器的结论: 1. 从kafka-manager的统计数据可以看出, 公司22服务器\(12C 48G\)的硬件条件下, 单个broker在处理速率上线为50MB/S. 2. 消息长度尽量精简 3. 尽量使用异步发送模式, 生产者属性max.block.time视情况设长些 4. 非重要消息, 参数acks设为0\(无需服务端确认\) 5. max.block.ms设根据实际设长一些

## 三. 单机服务器\(1.22\) – 同步提交模式

其他条件与异步提交模式一致, 但提交模式设置为同步, 只在一台producer的情况下, 如消息内容100Byte,一般响应时间是11,12ms

![](../../../.gitbook/assets/4039130-5f5775bf2aa795f1.png)

消息内容增加到1000Byte, 正常响应时间提高到差别不大ms ![](../../../.gitbook/assets/4039130-fd0ad4cfc5a0bfd3.png)

消息内容增加到128K, 响应时间大多在到30-70ms, topic带宽占用50M, 每秒消息数100-200条之间. ![](../../../.gitbook/assets/4039130-c99fbb0b6b62e86a.png) ![](../../../.gitbook/assets/4039130-11b6bab82d7f2ed1.png)

同步超时timeout=100ms, 提交100w条消息100Byte的消息, 如下图kafka-manager统计数据显示,吞吐很低. 观察日志, 处理耗时接近12分钟. 响应时间大于50ms小于100ms的消息1921条, 大于100ms消息数1408条, 提交失败0条. ![](../../../.gitbook/assets/4039130-6793a003d37b63a2.png)

结论: 1. 同步模式建议在数据量少, 能接受比较长响应时间, 并且需要获得消息保存的元数据\(topic,分区,offset等\)才使用. 若设置了超时时间, 等待返回若发生超时, 不一定是提交失败.

```text
Future<RecordMetadata> future = producer.send(…);
//设置等待返回超时时间
RecordMetadata metadata = future.get(100, TimeUnit.MILLISECONDS);
```

## 四. 单机服务器\(1.22\) – 消费

topic分3个分区, 单消费者拉取消息，kafka-manager显示数据流出速率稳定在25MB/S左右. ![](../../../.gitbook/assets/4039130-6fb10119e4349afa.png)

topic分3个分区, 三个消费者同时拉取数据, kafka-manager数据流出速率稳定在47MB/S左右, 接近异步提交数据的满载50MB/S. ![](../../../.gitbook/assets/4039130-d1e680fb20e4b8b3.png)

topic分1个分区, 单消费者拉取消息，kafka-manager显示数据流出速率也稳定在25MB/S左右. ![](../../../.gitbook/assets/4039130-93f2b0db7fe48c9c.png)

可以看出, 1\) 单个消费者的拉取速率上线是25MB/S 2\) 在多分区情况下, 多消费者的总读取速率会比单消费者高. 但受限于单服务器节点, 读取速率并不会成倍增长.

## 五. 小集群服务器\(1.20, 1.21,1.22\)硬件信息

参考第二章节

都是同样配置的机器

## 六. 小集群3个broker – 异步提交模式

### 7.1 生产者客户端数据

#### 6.1.1 参数与部署

生产者关键参数:

```text
acks=1
batch.size=1024*128
linger.ms=10
max.block.ms=10000
```

生产者部署: 分别部署9台, 其中3台物理机器\(1.20, 1.21, 1.23\)以及6台JVM虚拟机\(1.129, 1.113, 1.136, 1.28, 1.109, 1.112\), coreSize=16的线程池分90000次发送一批1000条_100Byte的数据, 总共发送81kw_100Byte条消息.

三台物理机器配置跟1.22服务器一致, 并且内网传输速度实测超过100MB/s, KVM虚拟机磁盘读写、cpu核心数、内存频率大小、网络传输速度等都比物理机低.

#### 6.1.2 生产者端

提交的81kw\*100Byte条消息没有丢失一条. ![](../../../.gitbook/assets/4039130-d67e69b7d86ac748.png)

第二次执行 ![](../../../.gitbook/assets/4039130-5d76fc54dea83fe3.png)

### 6.2 kafka服务端数据

#### 6.2.1 CPU内存情况

1.20 broker服务器 ![](../../../.gitbook/assets/4039130-16960ce09305bfd1.png)

1.21 broker服务器 ![](../../../.gitbook/assets/4039130-39d6cb4ccaee780d.png)

1.22broker服务器 ![](../../../.gitbook/assets/4039130-013c449b652633c8.png)

#### 6.2.2 kafka-manager

由于这里是集群, 能看到集群汇总数据的只有kafka-manager, 因此这里只贴出kafka-manager的截图 ![](../../../.gitbook/assets/4039130-ad70be2740340db4.png)

## 七. 小集群3个broker – 同步提交模式

参考单机同步提交模式.

## 八. 小集群3个broker – 消费

由于由于topic设置为3个分区, 并且一个分区只能由一个消费者集群中的一个消费者实例读取\(防止重复读取\), 需要起3个消费者.

9.1 一组三个消费者同时拉取消息. ![](../../../.gitbook/assets/4039130-5ac5906e25033749.png)

9.2 两组三个消费者同时拉取消息. ![](../../../.gitbook/assets/4039130-a32ef691f87d2d1c.png)

分区与服务器分配关系 ![](../../../.gitbook/assets/4039130-4060f97a39f69847.png)

观察发现20broker-\[0\]分区的消息消费速度比其他分区的快非常多. ![](../../../.gitbook/assets/4039130-4a529d1de2924fc5.png) ![](../../../.gitbook/assets/4039130-40cbfe7bf90a1936.png)

broker的CPU负载

1.20 – controller ![](../../../.gitbook/assets/4039130-d59ab6236f075cc9.png)

1.21 ![](../../../.gitbook/assets/4039130-01521926a9252137.png)

1.22 ![](../../../.gitbook/assets/4039130-bc6bd555c6daf4ec.png)

由上述图片观察到: 1分区消息消费速度最慢, 1分区所分配的broker-22吞吐最低\(48M\), 但是CPU占用率远远高于其他两个broker.

运维同学帮忙排查了服务器, 通过iostate观察磁盘读写指标, 发现下图后三项数值比其他两台服务器高很多, 表明磁盘每次io操作耗时长, 大量CPU资源花在磁盘io操作, 有可能是磁盘性能比其他两台低, 再加上22服务器部署了很多其他应用, 空余内存也20,21少, 建议使用配置相同的干净的机器进行测试. 这也从侧面反映出kafka服务器对磁盘的依赖高于CPU跟内存.

### ![](../../../.gitbook/assets/4039130-b1da2a9ae83a934c.png)

## 九.最后

通过以上不够严谨的测试, 公司20,21,22服务器环境下的kafka关键吞吐指标如下 1. 数据输入\(写消息\)

* 单个生产者异步提交数据速率上限在37MB/S
* 单个broker的数据输入速率上限在50MB/S左右
* 集群的数据输入速率上限是单台broker速率的N倍\(N=集群broker数量\)
* 数据输出\(读取消息\)
* 单个消费者消费数据速度上限在25MB/S左右
* 单个broker的数据输出速度上限在50MB/S左右
* 集群的数据输出速率上限**接近**单台broker速率的N倍\(N=集群broker数量\)
* 单条消息的长度
* kafka并没有限制单条消息的长度, 但服务端有接受消息的最大长度限制`message.max.bytes= 1000000`, 生产者跟消费者对应的消息长度不能大于服务器这个配置值.
* 如果要把kafka封装为公共服务以便内部程序调用, 那么还需要内部通信组件数据传输长度限制, 公司使用的是dubbo, 默认长度是8M.
* 消息长度精良精简, 有助于提高发送跟接受的消息数量
* kafka对磁盘的依赖要高于CPU跟内存
* 建议使用多磁盘来存储数据日志文件以分散磁盘读写压力
* 如要部署在阿里云服务器, 需要注意一下内网传输带宽以及内网收发包数值限制

  ![](../../../.gitbook/assets/4039130-c4e9027fe96e61ce.png)

1. 关于丢失消息的事情

丢失数据会在两个步骤发生, **发送失败\(集群未收到\)**跟**集群收到消息但内部同步异常导致丢失**. 由于测试集群只有三台broker, 结构简单, 业务量也不好测, 因此没有发生内部同步异常丢失. 但发送失败发生过好几次\(见2.1.3\).

测试阶段发生的发送失败, 归根结底是broker的处理速度不够快, 以及生产者的消息生产速度远超过发送速度导致的, 可以从以下两方面缓解

* broker服务端 1.搭建集群, 多分区多broker分担请求流量, 多磁盘存储数据日志文件, 提高内存, 加快请求响应速度.
* 生产者 1.合理设置max.block.ms\(默认60s\), 这个设置收程序分配内存影响, 设置太长有可能导致内存不够分配而报异常. 2.增加程序分配的内存 3.精简消息长度, 能减少传输时间, 提高效率 4.尽量使用异步发送模式, producer参数acks设为0\(无需服务端确认\), 消息生产者不回调, 减少等待集群响应时间

