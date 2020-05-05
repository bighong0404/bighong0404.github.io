# zookeeper笔记

zookeeper是一个典型的分布式数据一致性解决方案. 基于它可以实现数据发布/订阅, 负载均衡, 命名服务, 分布式协调/通知, 集群管理, master选举, 分布式锁和分布式队列等功能.因为zookeeper能保证一下分布式一致性: 顺序一致性, 原子性, 单一视图, 可靠性, 实时性.

zookeeper将全量数据存储在内存中.

基本概念:

集群角色: Leader, Follower, Observer 三者都提供读服务, 但Observer不参与leader选举, 也不参与写操作的"过半写成功"策略, 只用以提升集群的读性能. Leader跟Follower是经过Leader选举过程决定的, 而Observer是在zoo.cfg中, 集群服务器列表写死配置的:

```text
server.1=ip2:2888:3888
server.2=ip3:2888:3888
server.3=ip4:2888:3888:observer
```

会话: 客户端与服务器之间保持一个TCP长连接, 心跳检测有效性.

数据节点: 持久节点, 临时节点

版本: 数据节点中的一个数据结构, 记录节点各项属性的版本

watcher, 允许用户在节点上注册watcher, 特定事件触发.

ACL\(Access Control List\)策略的节点权限控制, create, read, write, delete, admin

ZAB协议\(Zookeeper Atomic Broadcast\), zookeeper原子消息广播协议. 协议的核心是定义了事务请求的处理方式: 所有事务请求都交由全局唯一的Leader, 转换成一个事务提议\(Proposal\), 分发给Follower, 并等待Follower反馈. 但获得过半数Follower的正确反馈后, Leader再次给所有Follower分发Commit消息, 要求将之前的提议\(Proposal\)执行.

事务编号: 每个提议都有个64位的编号\(ZXID事务编号\), 高32位是主进程周期\(leader有效工作周期\)编号, 低32位是单调递增的事务编号

| Leader周期编号\(leader epoch\) | 本地单调递增编号 |
| :---: | :---: |
| 高32位 | 低32位 |

协议内容包括两种模式: 奔溃恢复和消息广播, 可以细分为三个阶段: 发现, 同步, 广播

zookeeper的应用场景: 发布/订阅, 负载均衡, 命名服务, 分布式协调/通知, 集群管理, Master选举, 分布式锁, 分布式队列等

