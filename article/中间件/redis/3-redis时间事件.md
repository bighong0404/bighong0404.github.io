Redis的时间事件分为以下两类：

- 定时事件：让一段程序在指定的时间之后执行一次。比如说，让程序X在当前时间的30毫秒之后执行一次。

- 周期性事件：让一段程序每隔指定时间就执行一次。

**Redis3.0版本只使用周期性事件，而没有使用定时事件。**



# 1. 实现

redis服务器将所**有时间事件都放在一个无序链表**中，每当时间事件执行器运行时，它就**遍历整个链表**，查找所有已到达的时间事件，并调用相应的事件处理器。



Redis服务器以**周期性事件**的方式来运行`redis.c/serverCron`函数, `serverCron`函数负的主要工作：

- 更新服务器的各类统计信息，比如时间、内存占用、数据库占用情况等。

- 清理数据库中的**过期键**值对。

- 关闭和清理连接失效的客户端。

- 尝试进行**AOF或RDB持久化**操作。

- 如果服务器是主服务器，那么对从服务器进行定期**同步**。

- 如果处于集群模式，对集群进行定期同步和连接测试。

redis3.0版本默认规定`serverCron`函数**每秒运行10次**，平均每间隔**100毫秒**运行一次。



# 2. `serverCron`函数

>  Redis服务器中的serverCron函数默认**每隔100毫秒执行一次**，这个函数负责管理服务器的资源，并保持服务器自身的良好运转。



## 2.1 更新服务器时间缓存

由于获取时间需要执行系统调用, 为了减少系统调用. redis缓存了`unixtime`属性和`mstime`属性被用作当前时间的缓存. 

```c
struct redisServer {
    // ...
    // 保存了秒级精度的系统当前UNIX时间戳
    time_t unixtime;
    // 保存了毫秒级精度的系统当前UNIX时间戳
    long long mstime;
    // ...
};
```

每100毫秒一次的频率更新unixtime属性和mstime属性，所以这两个属性记录的时间的精确度并不高. 

- 这两个属性只使用在打印日志、更新服务器的LRU时钟、决定是否执行持久化任务、计算服务器上线时间（uptime）这类对时间精确度要求不高的功能上。

- 对于为键设置过期时间、添加慢查询日志这种**需要高精确度时间**的功能来说，服务器还是会再次执行**系统调用**获得最准确的系统时间。



## 2.2 更新LRU时钟

```c
struct redisServer {
    // ...
    // 默认每10秒更新一次的时钟缓存，用于计算键的空转（idle）时长。
    unsigned lruclock:22;
    // ...
};
```

服务器的`lruclock`属性辅助计算键的空转时间, 这也是个模糊值. 

命令`OBJECT IDLETIME <key>`就是拿`lruclock`减去键的lru时间得出空转时间.



## 2.3 更新服务器每秒执行命令次数

`serverCron`函数中的`trackOperationsPerSecond`函数会以每100毫秒一次的频率执行，这个函数的功能是以抽样计算的方式，估算并记录服务器在最近一秒钟处理的命令请求数量**，**这个值可以通过`INFO status`命令的`instantaneous_ops_per_sec`域查看：

```shell
redis> INFO stats
# Stats
...
instantaneous_ops_per_sec:6  # 在最近的一秒钟内，服务器处理了大概六个命令
...
```



## 2.4 更新服务器内存峰值记录

每次serverCron函数执行时, 都会比对内存使用情况, 并保存**内存峰值大小**到服务器状态中的`stat_peak_memory`属性.


```c
struct redisServer {
    // ...
    // 已使用内存峰值
    size_t stat_peak_memory;
    // ...
};
```

`INFO memory`命令的`used_memory_peak`和`used_memory_peak_human`两个域分别以两种格式记录了服务器的内存峰值：

```shell
redis> INFO memory
# Memory
...
used_memory_peak:501824
used_memory_peak_human:490.06K
...
```



## 2.5 **处理SIGTERM信号**



服务器在接到`SIGTERM`信号之后,  `SIGTERM`信号处理器会把服务器状态的`shutdown_asap`标识为1.

每次serverCron函数运行时，会检查服务器状态的`shutdown_asap`属性值, 来决定是否关闭服务器：

```c
struct redisServer {
    // ...
    // 关闭服务器的标识：
    // 值为1时，关闭服务器，
    // 值为0时，不做动作。
    int shutdown_asap;
    // ...
};
```



## 2.6 **管理客户端资源**

**serverCron函数每次执行都会调用clientsCron函数**:

- 清理超时的客户端连接

- 释放客户端当前的输入缓冲区，并重新创建一个默认大小的输入缓冲区，防止客户端的输入缓冲区耗费了过多的内存。



## 2.7 管理数据库资源

serverCron函数每次执行都会调用databasesCron函数，

- 删除部分过期键
- 对字典进行收缩操作



## 2.8 执行被延迟的BGREWRITEAOF

在redis执行`BGSAVE`期间,  客户端发来`BGREWRITEAOF`命令, 会被延迟执行, 就是标记了服务器的`aof_rewrite_scheduled`属性为1. 

```c
struct redisServer {
    // ...
    // 如果值为1，那么表示有 BGREWRITEAOF命令被延迟了。
    int aof_rewrite_scheduled;
    // ...
};
```

每次serverCron函数执行时, 
- 当前没有执行`BGSAVE`或者`BGREWRITEAOF`
- aof_rewrite_scheduled属性的值为1，

那么服务器就会执行之前被推延的BGREWRITEAOF命令。





## 2.9 检查持久化操作的运行状态

服务器状态使用`rdb_child_pid`属性和`aof_child_pid`属性记录执行`BGSAVE`命令和`BGREWRITEAOF`命令的子进程的ID，这两个属性也可以用于检查BGSAVE命令或者BGREWRITEAOF命令是否正在执行：

```c
struct redisServer {
    // ...
    // 记录执行BGSAVE命令的子进程的ID：
    // 如果服务器没有在执行BGSAVE，
    // 那么这个属性的值为-1。
    pid_t rdb_child_pid;                /* PID of RDB saving child */
    // 记录执行BGREWRITEAOF命令的子进程的ID：
    // 如果服务器没有在执行BGREWRITEAOF，
    // 那么这个属性的值为-1。
    pid_t aof_child_pid;                /* PID if rewriting process */
    // ...
};
```



## 2.10 将AOF缓冲区中的内容写入AOF文件

若开启了了AOF功能, serverCron函数会调用相应的程序，将AOF缓冲区中的内容写入到AOF文件里面.



## 2.11 关闭异步客户端

**在这一步，服务器会关闭那些输出缓冲区大小超出限制的客户端**



## 2.12 增加cronloops计数器的值

服务器状态的cronloops属性记录了serverCron函数执行的次数：

```c
struct redisServer {
    // ...
    // serverCron函数的运行次数计数器
    // serverCron函数每执行一次，这个属性的值就增一。
    int cronloops;
    // ...
};
```

cronloops属性目前在服务器中的唯一作用，就是在复制模块中实现“**每执行serverCron函数N次就执行一次指定代码**”的功能.

