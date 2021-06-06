# 常见问题



## 1、fork耗时导致高并发请求延时

**原因**	

​	生成RDB快照，AOF rewrite，耗费磁盘IO的过程，主进程fork子进程的时候，子进程是需要拷贝父进程的空间内存页表的，会耗费一定的时间的.

​		一般来说，如果父进程内存有1个G的数据，那么fork大概会耗费在20ms左右，如果是10G~30G，那么就会耗费20 * (10 ~ 30)，也就是几百毫秒. `指令info stats`中的`latest_fork_usec`，可以看到最近一次form的时长. 

​	redis单机QPS一般在几万，fork可能一下子就会拖慢几万条操作的请求时长，从几毫秒变成1秒.

**优化思路**

​		fork耗时跟redis主进程的内存有关系，一般控制redis的内存在10GB以内.

 

## 2、AOF的阻塞问题

**原因**

​		redis将数据写入AOF缓冲区，单独起线程做每秒一次的fsync操作. 但是redis主线程会检查两次fsync的时间间隔，如果距离上次fsync时间超过了2秒，那么写请求就会阻塞.`appendfsync everysec`，最多丢失2秒的数据. 一旦fsync超过2秒的延时，整个redis就被拖慢.

**优化思路**

​		优化硬盘写入速度，建议采用SSD，大幅度提升磁盘读写的速度

 

## 3、主从复制延迟问题 

**处理思路**

主从复制可能会超时严重，这个时候需要良好的监控和报警机制.

在`info replication`中，可以看到master和slave复制的offset，做一个差值就可以看到对应的延迟量. 

可以通过shell脚本发送命令, 解析结果来计算差值, 如果延迟过多，那么就进行报警. 

 

## 4、主从复制风暴问题

**原因**

​		一下子让多个slave从master去执行全量复制，一份大的rdb同时发送到多个slave，会导致网络带宽被严重占用. 

**优化方案**

​		个master要挂载多个slave，那尽量用树状结构，不要用星型结构



# 内核参数调优

## 1、vm.overcommit_memory

0: 检查有没有足够内存，没有的话申请内存失败

1: 允许使用内存直到用完为止

2: 内存地址空间不能超过swap + 50%

 

如果是0的话，可能导致类似fork等操作执行失败，申请不到足够的内存空间

 

cat /proc/sys/vm/overcommit_memory

echo "vm.overcommit_memory=1" >> /etc/sysctl.conf

sysctl vm.overcommit_memory=1

 

## 2、swapiness

cat /proc/version，查看linux内核版本

如果linux内核版本<3.5，那么swapiness设置为0，这样系统宁愿swap也不会oom killer（杀掉进程）

如果linux内核版本>=3.5，那么swapiness设置为1，这样系统宁愿swap也不会oom killer



保证redis不会被杀掉

 

echo 0 > /proc/sys/vm/swappiness

echo vm.swapiness=0 >> /etc/sysctl.conf

 

## 3、最大打开文件句柄

ulimit -n 10032 10032



## 4、tcp backlog

 backlog的定义是已连接但未进行accept处理的SOCKET队列大小. 

nginx的backlog值也是511. 



cat /proc/sys/net/core/somaxconn

echo 511 > /proc/sys/net/core/somaxconn