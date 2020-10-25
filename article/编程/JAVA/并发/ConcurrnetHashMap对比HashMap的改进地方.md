表结构类似



1. Node[] tab的值, 除了null, `Node`, `TreeBin`, 还有`ForwardingNode`, `ReservationNode`
2. 使用分段锁,  在Node[] tab的head上锁
3. 比对, 查找, update操作都是cas操作, update操作都是double check操作