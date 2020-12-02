# 1. 原理图

![这里写图片描述](img/20171020172529956)

- ThreadLocal的核心功能都是有内置类ThreadLocalMap完成. 每个线程都有一个属性 `ThreadLocalMap threadLocals`,  用于维护线程隔离的数据副本.

  ```java
  /* ThreadLocal values pertaining to this thread. This map is maintained
   * by the ThreadLocal class. */
  ThreadLocal.ThreadLocalMap threadLocals = null;
  ```

- ThreadLocal对象创建时候都会为内置属性`threadLocalHashCode`赋值,   此hashcode帮助tl对象在ThreadLocalMap中作为Entry[]中能更均匀地分布.

  ```java
  private final int threadLocalHashCode = nextHashCode();
  
  private static AtomicInteger nextHashCode = new AtomicInteger();
  /**
   * 每创建一个ThreadLocal对象，ThreadLocal.nextHashCode 这个值就会增长 0x61c88647。
   * 这个值是斐波那契数,也叫黄金分割数。hash增量为这个数字，带来的好处就是 hash分布非常均匀。
   */
  private static final int HASH_INCREMENT = 0x61c88647;
  
  private static int nextHashCode() {
      return nextHashCode.getAndAdd(HASH_INCREMENT);
  }
  ```

- ThreadLocalMap的是个自定义的HashMap, 也有内部Entry[]数组, 而Entry继承了弱引用.  Key值就是ThreadLocal对象.  tl对象存储在Entry[]的哪个索引, 由tl对象的`threadLocalHashCode`计算得出. 

  ```java
  static class Entry extends WeakReference<ThreadLocal<?>> {
      /** The value associated with this ThreadLocal. */
      Object value;
  
      Entry(ThreadLocal<?> k, Object v) {
          super(k);
          value = v;
      }
  }
  ```

- 但发生hash碰撞时候, ThreadLocalMap并不会像hashmap一样使用链表来处理. 逻辑如下: 

  - 若当前Entry[index]是null, 则直接保存

  - 若Entry非空, 判断Entry对象的key == 要设置的key, 那么替换value

  - 若Entry非空, 判断Entry对象的key是否已经null, 是则进入清理程序, 并把value设进去

  - 都不是, 那就寻找Entry[index+1]. 执行上述两步判断, 直到Entry[index]是null为止.

    

# 2. 重要的几个ThreadLocalMap方法

## 1) replaceStaleEntry 工作原理





  

## 2) expungeStaleEntry 清理过期数据

主要做两个事情: 从staleSlot位置开始, 1. 清理过期数据(value, entry置空), 2. 未过期数据重新hash到更合适的位置. 

```java
/**
 * 参数 staleSlot   table[staleSlot] 就是一个过期数据，以这个位置开始 继续向后查找过期数据，直到碰到 slot == null 的情况结束。
 */
private int expungeStaleEntry(int staleSlot) {
    //获取散列表
    Entry[] tab = table;
    //获取散列表当前长度
    int len = tab.length;

    // expunge entry at staleSlot
    //help gc
    tab[staleSlot].value = null;
    //因为staleSlot位置的entry 是过期的 这里直接置为Null
    tab[staleSlot] = null;
    //因为上面干掉一个元素，所以 -1.
    size--;

    // Rehash until we encounter null
    //e：表示当前遍历节点
    Entry e;
    //i：表示当前遍历的index
    int i;

    //for循环从 staleSlot + 1的位置开始搜索过期数据，直到碰到 slot == null 结束。
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        //进入到for循环里面 当前entry一定不为null
        //获取当前遍历节点 entry 的key.
        ThreadLocal<?> k = e.get();
        //条件成立：说明k表示的threadLocal对象 已经被GC回收了... 当前entry属于脏数据了...
        if (k == null) {
            //help gc
            e.value = null;
            //脏数据对应的slot置为null
            tab[i] = null;
            //因为上面干掉一个元素，所以 -1.
            size--;
        } else {
            //执行到这里，说明当前遍历的slot中对应的entry 是非过期数据
            //因为前面有可能清理掉了几个过期数据。
            //且当前entry 存储时有可能碰到hash冲突了，往后偏移存储了，这个时候 应该去优化位置，让这个位置更靠近 正确位置。
            //这样的话，查询的时候 效率才会更高！

            //重新计算当前entry对应的 index
            int h = k.threadLocalHashCode & (len - 1);
            //条件成立：说明当前entry存储时 就是发生过hash冲突，然后向后偏移过了...
            if (h != i) {
                //将entry当前位置 设置为null
                tab[i] = null;
                //h 是正确位置。
                //以正确位置h 开始，向后查找第一个 可以存放entry的位置。
                while (tab[h] != null)
                    h = nextIndex(h, len);
                //将当前元素放入到 距离正确位置 更近的位置（有可能就是正确位置）。
                tab[h] = e;
            }
        }
    }
    return i;
}
```



## 3) cleanSomeSlots 清理工作





## 4) rehash()

1. 先全盘清理一波
2. 判断是否达到resize()的条件, `size >= - threshold / 4`
3. resize(), 新Entry[]容量翻倍, 旧Entry[]一个个重新hash保存到新Entry[]中, 遇到hash碰撞则往index+1尝试