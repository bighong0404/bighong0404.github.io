# zookeeper技术内幕

## 1. 系统模型

### 1.1 数据模型,

ZNode, 树形结构, "/"分割路径 事务Id, zxid全局唯一, 每次事务请求分配一个zxid

### 1.2 节点特征

* 类型: 持久节点\(persistent\), 创建后, 必须手动才能删除 持久顺序节点\(persistent\_sequential\), 父节点维护第一级子节点的顺序, 给子节点名加数字后缀 临时节点\(ephemeral\), 生命周期跟会话绑一起, 临时节点只能是叶子节点, 不能有下一级节点 临时顺序节点\(ephemeral\_sequential\), 跟持久顺序节点类似, 只不过是临时的
* 节点状态信息`get命令` ![&#x72B6;&#x6001;&#x4FE1;&#x606F;.png](https://upload-images.jianshu.io/upload_images/4039130-235f64208f3d4f5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.3 版本 - 乐观锁, 保证分布式数据原子性操作

version: 当前数据节点 **数据内容** 的版本号 cversion: 当前数据节点 **子节点** 的版本号 aversion: 当前数据节点 **ACL变更版本号** 的版本号

![image.png](https://upload-images.jianshu.io/upload_images/4039130-20c7d117ef3ca1e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

版本值表示的对应信息的修改次数\(从0起计\), 内容不变但又修改操作, 同样也会递增.

### 1.4 Watcher - 数据变更通知

* watcher时间的通知状态与时间类型 ![image.png](https://upload-images.jianshu.io/upload_images/4039130-5cb4411dde12e38e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 回调方法process\(\)

  > abstract public void process\(WatchedEvent even\);

`WatchedEvent`对象封装了3个基本属性: 通知状态`keeperState`, 时间类型`eventType`, 节点路径`path`.

另外, `WatcherEvent`是`WatchedEvent`调用`getWrapper()`方法把自己包装成一个可序列化的`WatcherEvent`事件, 以便通过网络传输到客户端. 客户端接收到这个事件对象后, 会还原成一个`WatchedEvent`事件, 调用`process(WatchedEvent even)`执行处理

* 工作机制

  可以概括为三个过程: **客户端注册Watcher**, **服务端处理Watcher**, **客户端回调Watcher**

![image.png](https://upload-images.jianshu.io/upload_images/4039130-6959c83af628124e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 客户端注册watcher
* 服务端处理Watcher
* 客户端回调Watcher

  SendThread接受事件通知

  EventThread处理事件通知

### 1.5 ACL\(Access Control List\) - 数据安全

| 权限模式 | 授权对象 |
| :--- | :--- |
| IP | IP地址或IP网段 |
| Digest | 通常是username:BASE64\(SHA1\(username:password\)\), 也可以自定义 |
| World | 只有一个: anyone |
| Super | Digest模式的特殊形式, 超级用户, 拥有所有权限, 在zoo.cfg配置 |

> dubbo在zk中的权限应用: [https://yq.aliyun.com/articles/284281?utm\_content=m\_37169](https://yq.aliyun.com/articles/284281?utm_content=m_37169)

## 2. 序列化与协议

### 2.1  zk使用Jute进行序列化

### 2.2 通信协议

#### 2.2.1 请求头\(请求连接/会话创建请求是无请求头\), 请求体

#### 2.2.1 响应头\(会把请求的xid原值返回\), 响应体

### 2.3 客户端

`zookeeper实例: 客户端入口` `ZKWatchManager: 客户端Watcher管理器` `HostProvider: 服务端地址列表管理器` `ClientCnxn: 客户端核心线程`

#### 2.3.1 创建会话

#### 2.3.2 服务端地址列表解析,获取服务器, 拓展

#### 2.3.3 ClientCnxn

核心线程, 内部包含2个线程: `SendThread: 网络I/O类线程, 负责客户端与服务端间的网络I/O通信` `EventThread: 事件线程, 负责对服务端事件回调处理`

#### 2.3.4 会话

**2.3.4.1 会话状态**

`Connecting, Connected, (ReConnecting, ReConnected), Closed`

**2.3.4.2 会话创建**

**2.3.4.3 会话管理\(心跳检测 + 分桶策略\)**

**2.3.4.4 会话清理**

#### 2.3.5 服务器启动

**2.3.5.1 单机版服务器启动**

**2.3.5.2 集群版服务器启动**

* Leader选举 Leader选举是保证分布式数据一致性的关键所在。当Zookeeper集群中的一台服务器出现以下两种情况之一时，需要进入Leader选举。 _**服务器启动时期的Leader选举**_ --- 当两台机启动了, 就进入选举流程
* 初始情况都会把自己作为leader来投票, 投票信息\(包含SID也就是myid, ZXID\)发给其他服务器实例
* 接收到其他实例的投票信息, 检查有效性: 是否本轮投票, 是否来自LOOKING状态的服务器
* 处理投票, PK规则

  `1)  对比zxid(高32位是主进程周期"leader有效工作周期""编号, 低32位是单调递增的事务编号), 大的优先作为Leader建议`

  `2)  zxid相同, myid大的作为Leader建议`

  `3)  实例变更投票信息, 再一次向其他实例发送投票信息`

* 统计投票, 实例接收到过半的其他机器投来相同的投票信息, 则认为投票信息对应的SID机器就是Leader
* 修改服务器状态, Follower服务器修改为Following, Leader服务器修改为Leading

_**服务器运行期间的Leader选举**_ 在zk集群正常运行过程中, 一旦选出一个Leader, 一般所有服务器角色不会再发生改变, Leader会一直是Leader, 即使Follower挂了或者新机器加入也不会影响Leader. 但是一旦Leader挂了, 那么整个集群将停止对外服务, 进入新一轮Leader选举. 1. 变更状态, 所有非Observer服务器把自己状态改为LOOKING 2. 生成投票信息并发给其他服务器. 由于是集群运行期间发起的Leader选举, 每个服务器上的zxid可能不一致. 3. 接受其他服务器的投票信息, 检查有效心性 4. 处理投票信息, pk 5. 统计投票, 选出Leader 6. 修改服务器状态

![&#x9009;&#x4E3E;&#x793A;&#x4F8B;&#x56FE;.png](https://upload-images.jianshu.io/upload_images/4039130-31702f627b8def33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

_**选举小结**_ 简单来说, 哪台服务器上的数据越新, 就越有可能成为Leader, 因为ZXID比较大, 越能保证数据的恢复. SID越大也一样比较可能成为Leader

* Leader选举算法分析与实现细节

  _**算法**_

  三种算法, 可以在zoo.cfg使用electionAlg属性来指定, 0~3总共4种用法

* 0表示纯UDP实现的LeaderElection算法
* 1表示UDP版本的FastLeaderElection算法, 非授权模式
* 2表示UDP版本的FastLeaderElection算法, 授权模式
* 3表示TCP版本的FastLeaderElection算法

  zookeeper3.4.0版本后废弃了前三种, 只留第四种TCP版本的FastLeaderElection算法

  ```text
  org.apache.zookeeper.server.quorum.FastLeaderElection
  ```

  ![&#x9009;&#x4E3E;&#x7B97;&#x6CD5;.png](https://upload-images.jianshu.io/upload_images/4039130-f6b217b7afa25ad1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

_**服务器状态**_ LOOKING, FOLLOWING, LEADING, OBSERVING

_**投票数据结构**_

```text
org.apache.zookeeper.server.quorum.Vote
```

id：被推举的Leader的SID。 zxid：被推举的Leader事务ID。 electionEpoch：逻辑时钟，用来判断多个投票是否在同一轮选举周期中，该值在服务端是一个自增序列，每次进入新一轮的投票后，都会对该值进行加1操作。 peerEpoch：被推举的Leader的epoch。 state：当前服务器的状态。 ![Vote.png](https://upload-images.jianshu.io/upload_images/4039130-f36e0cbdde0c76de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* Leader 和 Follower启动交互过程

> SASL认证: [http://www.nosqlnotes.com/technotes/zookeeper-digest-md5/](http://www.nosqlnotes.com/technotes/zookeeper-digest-md5/)

