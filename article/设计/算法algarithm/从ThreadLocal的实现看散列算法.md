 JDK 的 ThreadLocal 源码有一段有意思的代码，如下所示。

```java
private final int threadLocalHashCode = nextHashCode();

/**
 * The difference between successively generated hash codes - turns
 * implicit sequential thread-local IDs into near-optimally spread
 * multiplicative hash values for power-of-two-sized tables.
 */
private static final int HASH_INCREMENT = 0x61c88647;

/**
 * Returns the next hash code.
 */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

定义了一个魔法值 HASH_INCREMENT = 0x61c88647，对于实例变量 threadLocalHashCode，每当创建 ThreadLocal 实例时这个值都会getAndAdd(0x61c88647)。

0x61c88647 转化成二进制即为 1640531527，它常用于在散列中增加哈希值。上面的代码注释中也解释到：**HASH_INCREMENT 是为了让哈希码能均匀的分布在2的N次方的数组里。**

**那么 0x61c88647 是怎么起作用的呢？**



# 什么是散列？

ThreadLocal 使用一个自定的的 Map —— ThreadLocalMap 来维护线程本地的值。首先我们先了解一下散列的概念。

散列（Hash）也称为哈希，就是把任意长度的输入，通过散列算法，变换成固定长度的输出，这个输出值就是散列值。

在实际使用中，不同的输入可能会散列成相同的输出，这时也就产生了冲突。通过上文提到的 HASH_INCREMENT 再借助一定的算法，就可以将哈希码能均匀的分布在 2 的 N 次方的数组里，保证了散列表的离散度，从而降低了冲突几率.

哈希表就是将数据根据散列函数 f(K) 映射到表中的特定位置进行存储。因此哈希表最大的特点就是可以根据 f(K) 函数得到其索引。

HashMap 就是使用哈希表来存储的，并且采用了链地址法解决冲突。

简单来说，哈希表的实现就是数组加链表的结合。在每个数组元素上都一个链表结构，当数据被 Hash 后，得到数组下标，把数据放在对应下标元素的链表上。

# 散列算法

先来说一下散列算法。散列算法的宗旨就是：构造冲突较低的散列地址，保证散列表中数据的离散度。常用的有以下几种散列算法：

**除法散列法**

散列长度 m, 对于一个小于 m 的数 p 取模，所得结果为散列地址。对 p 的选择很重要，一般取素数或 m

公式：f(k) = k % p （p<=m）

因为求模数其实是通过一个除法运算得到的，所以叫“除法散列法”

**平方散列法（平方取中法）**
先通过求关键字的平方值扩大相近数的差别，然后根据表长度取中间的几位数作为散列函数值。又因为一个乘积的中间几位数和乘数的每一位都相关，所以由此产生的散列地址较为均匀。

公式：f(k) = ((k * k) >> X) << Y对于常见的32位整数而言，也就是 f(k) = (k * k) >> 28

**斐波那契（Fibonacci）散列法**
和平方散列法类似，此种方法使用斐波那契数列的值作为乘数而不是自己。

对于 16 位整数而言，这个乘数是 40503。

对于 32 位整数而言，这个乘数是 2654435769。

对于 64 位整数而言，这个乘数是 11400714819323198485。

>具体数字是怎么计算得到的下文有介绍。
>
>为什么使用斐波那契数列后散列更均匀，涉及到相关数学问题，此处不做更多解释。


为什么使用斐波那契数列后散列更均匀，涉及到相关数学问题，此处不做更多解释。

公式：f(k) = ((k * 2654435769) >> X) << Y对于常见的32位整数而言，也就是 f(k) = (k * 2654435769) >> 28

这时我们可以隐隐感觉到 0x61c88647 与斐波那契数列有些关系。

**随机数法**
选择一随机函数，取关键字的随机值作为散列地址，通常用于关键字长度不同的场合。

公式：f(k) = random(k)

**链地址法（拉链法）**
懂了散列算法，我们再来了解下拉链法。拉链法是为了 HashMap 中降低冲突，除了拉链法，还可以使用开放寻址法、再散列法、链地址法、公共溢出区等方法。这里就只简单介绍了拉链法。

把具有相同散列地址的关键字(同义词)值放在同一个单链表中，称为同义词链表。有 m 个散列地址就有 m 个链表，同时用指针数组 T[0..m-1] 存放各个链表的头指针，凡是散列地址为 i 的记录都以结点方式插入到以 T[i] 为指针的单链表中。T 中各分量的初值应为空指针。

- HashMap：
  ![img](img/p-1591019001002)



- 除法散列（k=16）：

  ![img](img/p-1591019015966)

- 斐波那契散列：

![img](img/p-1591019027011)

可以看出用斐波那契散列法调整之后会比原来的除法散列离散度好很多。



# ThreadLocalMap 的散列

认识完了散列，下面回归最初的问题：0x61c88647 是怎么起作用的呢？

先看一下 ThreadLocalMap 中的 set 方法

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    ...
}
```



ThreadLocalMap 中 Entry[] table 的大小必须是 2 的 N 次方（len = 2^N）那 len-1 的二进制表示就是低位连续的 N 个 1， 那key.threadLocalHashCode & (len-1) 的值就是 threadLocalHashCode的低 N 位。

然后我们通过代码测试一下，0x61c88647 是否能让哈希码能均匀的分布在 2 的 N 次方的数组里。

public class MagicHashCode {
    private static final int HASH_INCREMENT = 0x61c88647;

```java
public static void main(String[] args) {
    hashCode(16); //初始化16
    hashCode(32); //后续2倍扩容
    hashCode(64);
}

private static void hashCode(Integer length){
    int hashCode = 0;
    for(int i=0; i< length; i++){
        hashCode = i * HASH_INCREMENT+HASH_INCREMENT;//每次递增HASH_INCREMENT
        System.out.print(hashCode & (length-1));
        System.out.print(" ");
    }
    System.out.println();
}
```
}
结果：

```
7 14 5 12 3 10 1 8 15 6 13 4 11 2 9 0 

7 14 21 28 3 10 17 24 31 6 13 20 27 2 9 16 23 30 5 12 19 26 1 8 15 22 29 4 11 18 25 0 

7 14 21 28 35 42 49 56 63 6 13 20 27 34 41 48 55 62 5 12 19 26 33 40 47 54 61 4 11 18 25 32 39 46 53 60 3 10 17 24 31 38 45 52 59 2 9 16 23 30 37 44 51 58 1 8 15 22 29 36 43 50 57 0 
```

产生的哈希码分布确实是很均匀，而且没有任何冲突。再看下面一段代码：

```java
public class ThreadHashTest {
    public static void main(String[] args) {
        long l1 = (long) ((1L << 32) * (Math.sqrt(5) - 1)/2);
        System.out.println("as 32 bit unsigned: " + l1);
        int i1 = (int) l1;
        System.out.println("as 32 bit signed:   " + i1);
        System.out.println("MAGIC = " + 0x61c88647);
    }
}
```



结果：

```
as 32 bit unsigned: 2654435769
as 32 bit signed:   -1640531527
MAGIC = 1640531527
```

| 16进制     | 10进制     | 2进制                            | 补码                             |
| :--------- | :--------- | :------------------------------- | :------------------------------- |
| 0x61c88647 | 1640531527 | 01100001110010001000011001000111 | 10011110001101110111100110111001 |

可以发现 `0x61c88647` 与一个神奇的数字产生了关系，它就是` (Math.sqrt(5) - 1)/2`。也就是传说中的黄金比例 0.618（0.618 只是一个粗略值），即0x61c88647 = 2^32 * 黄金分割比。同时也对应上了上文所提到的斐波那契散列法。

可以发现 0x61c88647 与一个神奇的数字产生了关系，它就是 (Math.sqrt(5) - 1)/2。也就是传说中的黄金比例 `0.618`（0.618 只是一个粗略值），即`0x61c88647 = 2^32 * 黄金分割比`。同时也对应上了上文所提到的斐波那契散列法。

# 黄金比例与斐波那契数列

最后再简单介绍一下黄金比例，这个概念我们经常能听到，又称黄金分割点。

黄金分割具有严格的比例性、艺术性、和谐性，蕴藏着丰富的美学价值，而且呈现于不少动物和植物的外观。现今很多工业产品、电子产品、建筑物或艺术品均普遍应用黄金分割，展现其功能性与美观性。

对于斐波那契数列大家应该都很熟悉，也都写过递归实现的斐波那契数列。

斐波那契数列又称兔子数列：

第一个月初有一对兔子

第二个月之后（第三个月初），它们可以生育

每月每对可生育的兔子会诞生下一对新兔子

兔子永不死去

转化成数学公式即：

```
f(n) = f(n-1) + f(n-2) (n>1)
...
f(0) = 0
f(1) = 1
```

当n趋向于无穷大时，前一项与后一项的比值越来越逼近黄金比

最后总结下来看，ThreadLocal 中使用了斐波那契散列法，来保证哈希表的离散度。而它选用的乘数值即是`2^32 * 黄金分割比`。
