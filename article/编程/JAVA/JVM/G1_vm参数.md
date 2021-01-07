**性能参数：**

| 参数名称                             | 含义                                                         | 默认值                                                    | 描述                                                         |
| ------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- | ------------------------------------------------------------ |
| 性能参数                             |                                                              |                                                           |                                                              |
| -Xms                                 | 初始堆大小                                                   | 物理内存的1/64(<1GB)                                      | 默认(MinHeapFreeRatio参数可以调整)空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制. |
| -Xmx                                 | 最大堆大小                                                   | 物理内存的1/4(<1GB)                                       | 默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到 -Xms的最小限制， 一般这两个值会设置成一致的，避免在GC完之后动态的调整堆大小，调整内存会消耗系统资源，对应用会有影响 |
| -Xss                                 | 每个线程的堆栈大小                                           | 1024K(1.8+)                                               | 线程栈大小意味着一个线程最多能压入多少栈帧，也就是不返回得执行多少方法， 一般不需要特殊调整，除非你的程序有部分类似递归调用这种很吃栈深度的逻辑，那么可以适当调整此参数的默认值 |
| -XX:NewRatio                         | 年轻代(包括Eden和两个Survivor区)与 年老代的比值(除去持久代)  | 2                                                         | Old/New的比值，默认值为2，表示Old代与Young代的比值为2比1， 使用G1时一般此参数不设置，由G1来动态的调整，逐渐调整至最优值 |
| -XX:SurvivorRatio                    | Eden区与Survivor区的大小比值                                 | 8                                                         | Eden/Survivor的比例，设置为8,则两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10 |
| -XX:MaxDirectMemorySize              | 直接内存大小                                                 | -Xmx                                                      | 直接内存非常重要，很多IO处理都需要直接内存参与，直接内存不被jvm管理，所以就不存在GC时内存地址的移动问题，直接内存会作为堆内存和内核之间的中转站 |
| -XX:PretenureSizeThreshold           | 大对象晋升老年代阈值                                         | Region的一半                                              | 大于此值的对象直接在老年代分配，这么做是为了预防新生代存在大量大对象，很容易被撑满 |
| -XX:MaxTenuringThreshold             | 新生代晋升老年代阈值                                         | 15                                                        | 晋升到老年代对象的年龄，每个对象在坚持过一次MinorGC后，年龄就增加1，当超过这个参数值时，就进入老年代 |
| -XX:MaxGCPauseMillis                 | 垃圾回收的最长时间(最大暂停时间)                             | 200ms                                                     | 设置GC最大的停顿时间，G1会尽量达到此期望值，如果GC时间超长，那么会逐渐减少GC时回收的区域，以此来靠近此阈值 |
| -XX:PreBlockSpin                     | 初始自旋次数                                                 | 10                                                        | 使用自旋锁时初始自旋次数，在此基础上上下浮动，生效前提开UseSpinning |
| -XX:RelaxAccessControlCheck          | 放开通过放射获取方法和属性时的许可校验                       | 默认关闭                                                  | 在Class校验器中，放松对访问控制的检查,作用与reflection里的setAccessible类似 |
| -XX:CompileThreshold                 | JIT预编译阈值                                                | 1000                                                      | 通过JIT编译器，将方法编译成机器码的触发阀值，可以理解为调用方法的次数，例如调1000次，将方法编译为机器码 |
| -XX:G1ReservePercent                 | G1为分配担保预留的空间比例                                   | 10                                                        | 预留10%的内存空间，应对新生代的分配担保情形                  |
| -XX:InitiatingHeap OccupancyPercent  | 启动并发GC时的堆内存占用百分比(InitiatingHeapOccupancyPercent) | 45                                                        | G1用它来触发并发GC周期,基于整个堆的使用率,而不只是某一代内存的使用比例。值为 0 则表示“一直执行GC循环)’. 默认值为 45 (例如, 全部的 45% 或者使用了45%). |
| -XX:G1HeapRegionSize                 | G1内堆内存区块大小                                           | (Xms + Xmx ) /2 / 2048 , 不大于32M，不小于1M，且为2的指数 | G1将堆内存默认均分为2048块，1M<region<32 M，当应用频繁分配大对象时，可以考虑调整这个阈值，因为G1的Humongous区域只能存放一个大对象，适当调整Region大小，尽量让其刚好超过大对象的两倍大小，这样就能充分利用Region的空间 |
| -XX:GCTimeRatio                      | GC时间占运行时间的比例                                       | G1为9，CMS为99                                            | GC时间占总时间的比例，默认值为99，即允许1%的GC时间。仅仅在Parallel Scavenge收集时有效，公式为1/(1+n) |
| -XX:G1HeapWastePercent               | 触发Mixed GC的可回收空间百分比                               | 5%                                                        | 在global concurrent marking结束之后，我们可以知道old gen regions中有多少空间要被回收，在每次YGC之后和再次发生Mixed GC之前，会检查垃圾占比是否达到此参数，只有达到了，下次才会发生Mixed GC |
| -XX:G1MixedGCLive ThresholdPercent   | 会被MixGC的Region中存活对象占比                              | 85%                                                       | old generation region中的存活对象的占比，只有小于此参数，才会被选入CSet |
| -XX:G1MixedGCCountTarget             |                                                              | 8                                                         | 一次global concurrent marking之后，最多执行Mixed GC的次数    |
| -XX:G1NewSizePercent                 |                                                              | 5%                                                        | 新生代占堆的最小比例                                         |
| -XX:G1MaxNewSizePercent              |                                                              | 60%                                                       | 新生代占堆的最大比例                                         |
| -XX:GCPauseIntervalMillis            |                                                              |                                                           | 指定最短多长可以进行一次gc                                   |
| -XX:G1OldCSetRegion ThresholdPercent | Mixed GC每次回收Region的数量                                 | 10%                                                       | 一次Mixed GC中能被选入CSet的最多old generation region数量比列 |
| -XX:ParallelGCThreads                |                                                              |                                                           | STW期间，并行GC线程数                                        |
| -XX:ConcGCThreads                    |                                                              | -XX:ParallelGCThreads/4                                   | 并发标记阶段，并行执行的线程数                               |

**功能开关：**

| 参数名称                     | 含义                       | 默认值                      | 描述                                                         |
| ---------------------------- | -------------------------- | --------------------------- | ------------------------------------------------------------ |
| -XX:+UseSpinning             | 是否使用适应自旋锁         | 1.6+默认开启                | 这是对synchronized的优化，当运行到synchronized的代码块时，会尝试自旋，如果在自旋期间获取到了锁，那么下次会逐渐的增加自旋时间，反之则减少自旋时间，当自旋时间减少到一定程度后，会关闭自旋机制一段时间，使用重量级锁 |
| -XX:+DisableExplicitGC       | 禁止System.gc()            | 默认启用(DisableExplicitGC) | 禁止在运行期显式地调用System.gc()                            |
| -XX:+UseG1GC                 | 使用G1做为GC收集器         | 1.8+                        | 在1.8+后默认使用，其在大内存情形下能够发挥更加卓越的性能，相较于其他GC最大的差异是其每次回收老年代时只会回收最值得回收的部分内存空间，而不是整个老年代 |
| -XX:+UseThreadPriorities     | 是否开启线程优先级         | 默认开启                    | java 中的线程优先级的范围是1～10，默认的优先级是5。“高优先级线程”会优先于“低优先级线程”执行，也就是竞争CPU时间片时，高优先级线程会被优待 |
| -XX:+UseBiasedLocking        | 是否开启偏向锁             | 默认开启                    | 和自旋锁一样，也是synchronized的优化措施之一，当synchronized代码块只被一个线程使过时，将偏向锁的ThreadID设置为自身线程ID，这样每次运行时只要比对ThreadID和自身一致就直接运行 |
| -XX:+HandlePromotionFailure  | 老年代是否允许分配担保失败 | DK1.6之后，默认开启         | 是否允许分配担保失败，即老年代的剩余最大连续空间不足以引用新生代的整个Eden和Survivor区的所有存活对象的极端情况（以此判断是否需要对老年代进行以此FullGC） |
| -XX:+UseTLAB                 | 使用线程本地分配缓冲区     | Server模式默认开启          | Thread Local Allocation Buffer，此区域位于Eden区。当多线程分配内存区块时，因为内存分配和初始化数据是不同的步骤，所以在分配时需要对内存区块上锁，由此会引发区块锁竞争问题。此参数会让线程预先分配一块属于自己的空间(64K-1M)，分配时先在自己的空间上分配，不足时再申请，这样能就不存在内存区块锁的竞争，提高分配效率 |
| -XX:+UseCompressedOops       | 是否开启指针压缩           | 32G内存下默认开启           | 开启指针压缩，则Object Head 里的Klass Pointer为4字节，复杂属性的引用指针为4字节，数组的长度用4字节表示，这样能够节省部分内存空间 |
| -XX:+PrintAdaptiveSizePolicy |                            |                             | 自适应策略，调节Young Old Size，一般G1不会设置新生代和老年代大小，而有G1根据停顿时间逐渐调整新生代和老年代的空间比例 |

**调试参数：**

| 参数名称                           | 含义                  | 默认值                         | 描述                                                         |
| ---------------------------------- | --------------------- | ------------------------------ | ------------------------------------------------------------ |
| -XX:HeapDumpPath                   | 堆内存快照到处路径    |                                | 指定HeapDump的文件路径或目录，使用前需开启HeapDumpOnOutOfMemoryError |
| -XX:+HeapDumpOnOutOfMemoryError    | 在OOM后导出堆内存快照 | 默认关闭                       |                                                              |
| -XX:OnError                        | 在Error时执行操作     |                                | 可配置相应命令，在Error时执行此命令，比如发送邮件            |
| -XX:OnOutOfMemoryError             | 在发生OOM时执行操作   |                                |                                                              |
| -XX:+PrintCommandLineFlags         | 打印jvm参数           |                                | 在程序运行前打印出用户手动设置或者JVM自动设置的XX选项        |
| -XX:+PrintGC                       | GC时打印概览信息      | 关闭                           |                                                              |
| -XX:+PrintGCDetails                | GC时打印详细信息      | 关闭                           |                                                              |
| -XX:+PrintGCTimeStamps             | 打印GC用时            | 关闭                           |                                                              |
| -XX:+PrintGCApplicationStoppedTime | 打印GC时应用暂停时间  | 关闭                           |                                                              |
| -Xloggc:< filename>                |                       | 将GC相关日志记录到文件以便分析 |                                                              |
| -XX:GCLogFileSize                  |                       | 8K                             | 使用前需开启-Xloggc，GC日志文件默认大小，当日志超过8K时需要被滚动存储 |
| -XX:+UseGCLogFileRotation          |                       | 关闭                           | 开启GC日志文件的滚动存储功能                                 |
| -XX:NumberOfGCLogFiles             | GC文件数量            | 1                              | GC日志滚动文件数量，超过时会删除最先创建的                   |
| -XX:+CITime                        |                       | 关闭                           | 打印启动时花费在JIT Compiler 时间                            |