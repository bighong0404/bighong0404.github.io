# 数据结构

HashMap的实现是Node{hash, key, value, next}结构的数组 + 链表 + 红黑树

![img](img/9bb15b36ca67499fa0970a497a20c149.jpeg)



putVal()发生hash碰撞时候, 会以链表形式追加, 若当前链表节点数超过8(`TREEIFY_THRESHOLD`), 并且整个HashMap的Node数达到64(`MIN_TREEIFY_CAPACITY`), 则当前追加的链表转为红黑树.



# 核心方法



## 保存方法 put() -> putVal()





## 扩容方法 resize() 



