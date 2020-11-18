

# 1. 处理多线程竞争的方式, 都是CAS操作, 再配合分段锁

LongAdder, ConcurrentHashMap思路都差不多



# 2. ThreadPoolExecutor

Doug Lea细心到令人发指!   Worker线程允许用户threadFactory创建, 可能存在有bug. 因此在两处地方做了判断.

 `addWorkder()`中如下几行代码:

```java
private boolean addWorker(Runnable firstTask, boolean core) {

  // ...一系列通过ctl判断线程池状态, workQueue, cas操作获取是否允许创建Worker
  
  // Worker w中持有的工作线程, 也就是Worker自己
  final Thread t = w.thread;
  // ★★★ 这里做非空判断, 是因为worker.thread是通过threadFactory.newThread(this)创建的, 而threadFactory允许用户自己实现,  Doug为了防止用户实现的线程工厂创建的thread有问题, 才做了空判断. 太细心了
  if (t != null) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      //...一些列二次检查
      
      // ★★★ 检查用户创建的线程是否已经被误启动了... 
      if (t.isAlive()) // precheck that t is startable
         throw new IllegalThreadStateException();
      
    } finally {
      mainLock.unlock();
    }
  }
}
```



