# 1. sentinal哨兵的介绍 

sentinal哨兵是redis集群架构中非常重要的一个组件，主要功能如下 

（1）集群监控，负责监控redis master和slave进程是否正常工作

（2）消息通知，如果某个redis实例有故障，那么哨兵负责发送消息作为报警通知给管理员

（3）故障转移，如果master node挂掉了，会自动转移到slave node上

（4）配置中心，如果故障转移发生了，通知client客户端新的master地址

 

哨兵本身也是分布式的，作为一个哨兵集群去运行，互相协同工作

（1）故障转移时，判断一个master node是宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题

（2）即使部分哨兵节点挂掉了，哨兵集群还是能正常工作的，因为如果一个作为高可用机制重要组成部分的故障转移系统本身是单点的，那就很坑爹了

 

# 2. 哨兵的核心知识

（1）哨兵至少需要3个实例，来保证自己的健壮性. 

（2）哨兵 + redis主从的部署架构，是不会保证数据零丢失的，只能保证redis集群的高可用性

（3）对于哨兵 + redis主从这种复杂的部署架构，尽量在测试环境和生产环境，都进行充足的测试和演练



# 3. 哨兵主备切换的数据丢失问题：异步复制、集群脑裂



## 3.1 异步复制的数据丢失

因为master -> slave的复制是异步的，所以可能有部分数据还没复制到slave，master就宕机了，此时这些部分数据就丢失了.

 

## 3.2 脑裂导致的数据丢失

​		某个master所在机器与其他slave节点的网络突然脱离(例如网络分区)，跟其他slave机器不能连接，但是实际上master还运行着.  此时哨兵可能就会认为master宕机了，然后开启选举，选其他slave切换成了master. 这个时候，集群里就会有两个master，也就是所谓的脑裂. 

​		此时虽然某个slave被切换成了master，但是可能client还没来得及切换到新的master，还继续写向旧master写数据.  等旧master恢复网络的时候，会被作为一个slave挂到新的master上去，自己的数据会清空，重新从新的master复制数据, 这时候之前网络异常期间写的数据就丢失了. 



## 3.3 解决方案

```shell
min-slaves-to-write n  # 要求至少有n个slave在线, n可以根据要求的最少slave数量进行修改

min-slaves-max-lag n  # slave数据复制和同步的延迟不能超过n秒
```

 配置上以上两个配置, 其中一个满足条件. master就不会再接收任何请求了,  可以减少异步复制和脑裂导致的数据丢失.

（1）**减少异步复制的数据丢失**

​		`min-slaves-max-lag`这个配置，就可以确保说，一旦slave复制数据和ack延时太长，就认为可能master宕机后损失的数据太多了，那么就拒绝写请求，这样可以把master宕机时由于部分数据未同步到slave导致的数据丢失降低的可控范围内. 

​		发生这种情况后, client可以选择把数据写到磁盘或者内存中, 等恢复后再写给master, 并且同时做限流, 减少数据涌入. 或者把数据发到mq. 定期取出来重新发给master. 

（2）**减少脑裂的数据丢失**

​		如果一个master出现了脑裂，跟其他slave丢了连接，那么上面两个配置可以确保说，如果不能继续给`min-slaves-to-write`台的slave发送数据，而且slave超过`min-slaves-max-lag`秒没有给自己ack消息，那么就直接拒绝客户端的写请求.  这样脑裂后的旧master就不会接受client的新数据，也就避免了过多数据丢失. 

​		因此在脑裂场景下，最多就丢失`min-slaves-max-lag`秒的数据.



# 4. 哨兵的核心底层原理深入解析

## 4.1 sdown和odown两种失败状态

 这两个状态是哨兵对master宕机的状态判断. 

- `sdown`是主观宕机. 一个哨兵ping一个master，超过了`is-master-down-after-milliseconds`指定的毫秒数之后，就主观认为master宕机.

- `odown`是客观宕机. `sdown`到`odown`转换的条件是,  一个哨兵在指定时间内收到了`quorum`数量的其他哨兵也认为那个master是sdown了，那么就认为是odown了，客观认为master宕机. 



## 4.2 哨兵集群的自动发现机制

​		哨兵互相之间的发现，是通过redis的pub/sub系统实现的. 每个哨兵都会往__sentinel__:hello这个channel里发送一个消息，这时候所有其他哨兵都可以消费到这个消息，以感知到其他的哨兵的存在. 

​		每隔两秒钟，每个哨兵都会往自己监控的某个master+slaves对应的__sentinel__:hello channel里发送一个消息，内容是自己的host、ip和runid还有对这个master的监控配置.

 		每个哨兵也会去监听自己监控的每个master+slaves对应的__sentinel__:hello channel，然后去感知到同样在监听这个master+slaves的其他哨兵的存在.

​		每个哨兵还会跟其他哨兵交换对master的监控配置，互相进行监控配置的同步. 



## 4.3 哨兵自动纠正slave配置

​		**哨兵会负责自动纠正slave的一些配置**. 

​		比如slave如果要成为潜在的master候选人，哨兵会确保slave在复制现有master的数据; 如果slave连接到了一个错误的master上，比如故障转移之后，那么哨兵会确保它们连接到正确的master上



## 4.4 slave->master选举算法

如果一个master被认为odown了，而且majority哨兵都允许了主备切换，那么某个哨兵就会执行主备切换操作，此时首先要从slave中选举一个slave来. 

slave提升为master的因素判断顺序:

- **slave优先级**, `slave-priority 100`, 数值越小优先级越高

- **复制offset**, offset越大, 说明复制的数据越多, 权重则越高
- **run id**,  取run id比较小的那个slave

- **跟master断开连接的时长**.   一个slave跟master断开连接已经超过了down-after-milliseconds的10倍，外加master宕机的时长，那么slave就被认为不适合选举为master



# 5. 哨兵架构部署

## 5.1 配置

`sentinel.conf`的最少配置

```shell
port 26379
bind 192.168.1.9
daemonize yes
logfile /var/log/redis-sentinal/26379
dir /var/redis-sentinal/26379

## 对master1的监控配置, 可以配置多组
sentinel monitor master1 <master1_ip> <master1_port> <quorum> # quaorum, master被认为客观宕机最少的票数
sentinel down-after-milliseconds master1 30000 # 超过多少毫秒跟master断了连接，哨兵就认为这个redis实例挂了
sentinel failover-timeout master1 60000  # 执行故障转移的timeout超时时长
sentinel parallel-syncs master1 1     # 新master被切换之后，一次同时有多少个slave被切换到去连接新master，重新做同步. 数字越低，花费的时间越多

## 对master2的监控配置
sentinel monitor master2 192.168.1.10 6379 2
sentinel down-after-milliseconds master2 30000
sentinel failover-timeout master2 60000
sentinel parallel-syncs master2 1 
```

## 5.2 启动哨兵

`redis-sentinel /etc/sentinal/26379.conf` 

通过日志可以看到哨兵之间的互相发现的过程, 

```log
6679:X 27 Dec 13:03:59.856 # +monitor master mymaster 192.168.1.9 6379 quorum 1 
6679:X 27 Dec 13:03:59.856 * +slave slave 192.168.1.9:6380 192.168.1.9 6380 @ mymaster 192.168.1.9 6379
6679:X 27 Dec 13:03:59.858 * +slave slave 192.168.1.9:6381 192.168.1.9 6381 @ mymaster 192.168.1.9 6379
```

也可以观察到哨兵切换master以及其他哨兵更新配置的过程, .

```log
6679:X 27 Dec 13:44:19.665 # +sdown master mymaster 192.168.1.9 6379 说明master服务已经宕机
6679:X 27 Dec 13:44:19.665 # +odown master mymaster 192.168.1.9 6379 #quorum 1/1
6679:X 27 Dec 13:44:19.665 # +new-epoch 1
6679:X 27 Dec 13:44:19.665 # +try-failover master mymaster 192.168.1.9 6379 开始恢复故障
6679:X 27 Dec 13:44:19.690 # +vote-for-leader 959ab7ad8eda390abfcf88f57ad718efe59f6dfc 1 投票选举哨兵leader，现在就一个哨兵所以leader就自己
6679:X 27 Dec 13:44:19.690 # +elected-leader master mymaster 192.168.1.9 6379 选中leader
6679:X 27 Dec 13:44:19.690 # +failover-state-select-slave master mymaster 192.168.1.9 6379
6679:X 27 Dec 13:44:19.767 # +selected-slave slave 192.168.1.9:6380 192.168.1.9 6380 @ mymaster 192.168.1.9 6379  选中其中的一个slave当做master
6679:X 27 Dec 13:44:19.767 * +failover-state-send-slaveof-noone slave 192.168.1.9:6380 192.168.1.9 6380 @ mymaster 192.168.1.9 6379  发送slaveof no one命令
6679:X 27 Dec 13:44:19.823 * +failover-state-wait-promotion slave 192.168.1.9:6380 192.168.1.9 6380 @ mymaster 192.168.1.9 6379 等待升级master
6679:X 27 Dec 13:44:20.139 # +promoted-slave slave 192.168.1.9:6380 192.168.1.9 6380 @ mymaster 192.168.1.9 6379 升级6380为master
6679:X 27 Dec 13:44:20.139 # +failover-state-reconf-slaves master mymaster 192.168.1.9 6379
6679:X 27 Dec 13:44:20.190 * +slave-reconf-sent slave 192.168.1.9:6381 192.168.1.9 6381 @ mymaster 192.168.1.9 6379
6679:X 27 Dec 13:44:21.184 * +slave-reconf-inprog slave 192.168.1.9:6381 192.168.1.9 6381 @ mymaster 192.168.1.9 6379
6679:X 27 Dec 13:44:21.185 * +slave-reconf-done slave 192.168.1.9:6381 192.168.1.9 6381 @ mymaster 192.168.1.9 6379
6679:X 27 Dec 13:44:21.239 # +failover-end master mymaster 192.168.1.9 6379 故障恢复完成
6679:X 27 Dec 13:44:21.239 # +switch-master mymaster 192.168.1.9 6379 192.168.1.9 6380主数据库从6379转变为6380
6679:X 27 Dec 13:44:21.239 * +slave slave 192.168.1.9:6381 192.168.1.9 6381 @ mymaster 192.168.1.9 6380  添加6381为6380的从库
6679:X 27 Dec 13:44:21.239 * +slave slave 192.168.1.9:6379 192.168.1.9 6379 @ mymaster 192.168.1.9 6380添加6379为6380的从库
6679:X 27 Dec 13:44:51.261 # +sdown slave 192.168.1.9:6379 192.168.1.9 6379 @ mymaster 192.168.1.9 6380 发现6379已经宕机，等待6379的恢复  注意+sdown 是+
————————————————
版权声明：本文为CSDN博主「牛在田野」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_40138160/article/details/78911864
```



## 5.3 哨兵管理命令

redis-cli连接上实例

``` shell
sentinel master master1 # 获取哨兵监听的master1的master信息
sentinel slaves master1 # 获取哨兵监听的master1的master, 对应的slave信息
sentinel sentinels master1 # 获取master1的所有哨兵信息
sentinel get-master-addr-by-name master1

sentinel reset master1 # 停止一个sentinal进程后, 或者下线了某个slave, 需要在其他sentinal进程执行次命令来重置master1状态
```



需要手动 `sentinel reset <master-group-name>`是哨兵模式容易漏操作的地方!!



## 5.4 注意事项

因为每个slave都有可能成为master, 因此所有节点都要配置连接密码配置

```shell
# redis.conf
requirepass xxxx 
masterauth xxxx  

# sentinal.conf
sentinel auth-pass <master-group-name> <pass>
```

