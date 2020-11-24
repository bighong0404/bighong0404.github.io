# ReentrantLock中公平锁与非公平锁的区别



1.  构造方法初始化时候, 指定了内部的`sync`对象是`FairSync`或者`NonfairSync`

   ```java
       /**
        * Creates an instance of {@code ReentrantLock}.
        * This is equivalent to using {@code ReentrantLock(false)}.
        */
       public ReentrantLock() {
           sync = new NonfairSync();
       }
   
       /**
        * Creates an instance of {@code ReentrantLock} with the
        * given fairness policy.
        *
        * @param fair {@code true} if this lock should use a fair ordering policy
        */
       public ReentrantLock(boolean fair) {
           sync = fair ? new FairSync() : new NonfairSync();
       }	
   ```

   

2. lock()操作本质上是执行`sync.lock()`,

   - `FairSync`或者`NonfairSyncd`在`lock()`的区别是, `FairSync`会直接执行`acquire(1)`, 而`NonfairSyncd`先争抢`state`, 失败了再执行`acquire(1)`;

     ```java
     /* FairSync */
     final void lock() {
     	acquire(1);
     }
     
     /* NonfairSyncd */
     final void lock() {
         if (compareAndSetState(0, 1))
             setExclusiveOwnerThread(Thread.currentThread());
         else
             acquire(1);
     }
     ```

     

   - `FairSync`或者`NonfairSyncd`在`tryAcquire(int acquires)`的区别, 先检查等待队列有没有线程在排队

     ```java
     /* FairSync */
     protected final boolean tryAcquire(int acquires) {
         final Thread current = Thread.currentThread();
         int c = getState();
         if (c == 0) {
             // hasQueuedPredecessors()检查是否等待队列中是否有线程在等待
             if (!hasQueuedPredecessors() &&
                 compareAndSetState(0, acquires)) {
                 setExclusiveOwnerThread(current);
                 return true;
             }
         }
         else if (current == getExclusiveOwnerThread()) {
             int nextc = c + acquires;
             if (nextc < 0)
                 throw new Error("Maximum lock count exceeded");
             setState(nextc);
             return true;
         }
         return false;
     }
     
     /* NonfairSyncd */
     protected final boolean tryAcquire(int acquires) {
         return nonfairTryAcquire(acquires);
     }
     final boolean nonfairTryAcquire(int acquires) {
         final Thread current = Thread.currentThread();
         int c = getState();
         if (c == 0) {
             // 没有调用hasQueuedPredecessors()
             if (compareAndSetState(0, acquires)) {
                 setExclusiveOwnerThread(current);
                 return true;
             }
         }
         else if (current == getExclusiveOwnerThread()) {
             int nextc = c + acquires;
             if (nextc < 0) // overflow
                 throw new Error("Maximum lock count exceeded");
             setState(nextc);
             return true;
         }
         return false;
     }
     ```

     

   

3.  // todo