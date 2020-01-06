---
title: ConcurrentHashMap源码解析
date: 2019-12-27 09:10:47
tags:
- 集合
categories:
- JAVA
toc: true
---


### 题外话

说来惭愧，倒腾了几个晚上，也没有完全明白其中的逻辑，只是对其底层组成和核心方法流程有了较深的了解，了解了多线程如何帮助扩容转移节点，这篇博客主讲JDK8的版本，希望对各位理解ConcurrentHashMap有一点帮助。

<br/>
## ConcurrentHashMap结构组成

ConcurrentHashMap 底层结构的实现上与 HashMap一样，也是基于 数组 + 链表 + 红黑树的结果
<!--more-->

  {% asset_img tree.jpg  结构图 %}

<br/>

不同的处是赋予了线程操作安全的属性，如何做到线程安全的呢？

------



### 基础结构

在介绍之前，必须先介绍其核心结构（我之前打算直接看核心方法的，，后面发现理解基础结构Node的字段对代码的理解和了解并发设计有很大帮助）

所以这里先介绍 节点Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    ....
}
```

<br/>

------



### 状态标志

这里主要关注这个属性字段  hash，如果是数组下标的头节点，那这个节点的hash值就不是我们认识的那个 32的hash了，而是变成了并发中用来表示线程操作的状态标志（你现在不需要理解，暂时记住，待会看put方法你就明白了）。

3种特殊的状态标志 ，当下标的节点（注意：是头节点！！） 

Node.hash = MOVED即 -1时代表正在扩容或者初始化，为 TREEBIN 即-2代表为 红黑树的根节点  为RESERVED 即-3时 这种源码中解释是用来做 占位符的标志

```java
static final int MOVED     = -1; // hash 表达转移节点
static final int TREEBIN   = -2; //  hash 表达 树的根节点
static final int RESERVED  = -3; // hash 表达暂时保留
```

所以只有 头节点的hash >= 0才代表该节点为 正常状态的链表节点 ！！

 <br/>

当然红黑树为了匹配多线程时的各种状态，它也和HashMap中的TreeNode的不一样，

是做了封装的TreeBin，它拥有了等待锁和持有读锁或写锁的状态标志。

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // values for lockState
    static final int WRITER = 1; // 设置当持有写锁的时候
    static final int WAITER = 2; // 设置当等待请求写锁的时候
    static final int READER = 4; // 读锁时自增值
    ....
}
```

<br/>

------



## 核心方法

### put

#### 方法源码

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ConcurrentHashMap 中 key value不允许为null
    if (key == null || value == null) throw new NullPointerException();
    //进行hash重新运算 (h ^ (h >>> 16)) & 0x7fffffff，官方解释 这种做法使得位扩散更合理，碰撞分布更均匀
    // 这个spread(hash)下来，保留了高16位，低16位进行了重新计算，然后又& HASHBIT使得最高位为0，其他位不变
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 当 tab为空,初始化table，该方法是做了线程同步CAS判断的，在下方有对initTable详细的解释
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //tabAt方法调用的是Unsafe.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE) 可以直接获取指定内存的数据，保证了每次拿到的数组下标数据都是最新的
        // 注意此时已经 f = 数组下标下头节点， i 为 数组下标
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 这里cas判断数组下标下 i 的头节点 f是否为空并赋值
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //此时说明 头节点f不为空 检查 f.hash == (MOVED为 -1) 内部移动，证明在扩容操作,待会看一下扩容时是否将节点hash重新赋值 MOVED标志位
        // 此时帮助还未转移的旧数组节点转移到新数组中
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else { 
            // 当数组不在扩容时， 数组下标下有节点时，synchronize锁住该下标的头结点，f在上面已经赋值 数组对应下标的链表头节点
            V oldVal = null;
             // 给当前数组下标的Node头结点加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // fh >=0 意味着 不在扩容 MOVED - 1，也不是红黑树 TREEBIN -2
                        binCount = 1;
                        // 在遍历链表时 ++bitCount，所以 bitCount代表链表的长度
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 判断节点是红黑树时，进行红黑树的插入操作
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 这一步做链表长度的判断，当>= 8时转换成红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 对当前容量大小进行检查，如果超过了阈值threshold（数组大小* 负载因子）就需要扩容
    addCount(1L, binCount);
    return null;
}
```

------



#### 流程总结

1、 先判断 传入的 key-value 是否为null，为null抛出异常

2、 spread方法对key重新hash，这样能保证分布到数组中更加的均匀

3 、在循环中进行判断

当tab为空时，进行初始化操作，默认数组大小16

否则 计算hash对应的下标 i，并获取下标对应头节点 f，当头节点为空时CAS直接插入链表节点；

否则判断头节点的 hash

4、当 hash = MOVED即-1，代表数组正在扩容或者初始化，此时让该线程帮助转移其他下标节点

5、其hash为其他状态时，用synchronized锁住当前头节点

通过头节点，遍历当前下标，用bitcount记录节点数

 **tabAt(tab, i) == f** 重新判断当前下标i下的头节点是否还是f，这里还是用到cas 

当hash标志位大于0，证明是正常状态下链表节点，直接遍历链表，插入到尾部

否则就是TreeBin 红黑树 ，进行树节点的判断、插入和旋转

6、此时判断bitCount大于等于8时，进行链表转换成红黑树

7、 addCount检查当前容量大小，当超过阈值时触发扩容 transfer



这里看一下上面的CAS判断，全部使用C++编写的 Unsafe类Native方法，确保直接从内存中取值，保证数据的一致性

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

<br/>

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```



因为PUT方法中使用到了initTable，这里先看InitTable方法

### initTable

介绍initTable方法前，必须先解释一个重要的参数 SizeCtl

```java
 /* 表初始化和大小调整控制。 如果为负，则表正在初始化或调整大小：-1用于初始化， else-（1 +活动的调整大小   *线程数）。
  * 除此以外，当table为null时，保留要使用的初始表大小创建，或默认为0。 
  * 初始化后，记录下一个要调整表大小的元素计数值。
  * 注意： 也就是说 大于0时 代表已经初始化了，记录的是下一次的扩容后的阈值
  */
  private transient volatile int sizeCtl;
```



```java
/ 使用sizeCtl中记录的大小初始化表。
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 从上面sizeCtl 解释可知，当 sizeCtl < 0 时，证明其他线程已在初始化数组，所以 thread.yield 让出当前线程执行权，保证只有一个线程在初始化数组
        if ((sc = sizeCtl) < 0)
            Thread.yield(); 
        // U是unSafe类,这里再判断一次如果 SIZECTL为0，证明无其他线程初始化tab, 设置 SIZECTL--,禁止其他线程修改tab大小，如果SIZECTL<0,则进行下次循环判断
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 这里为什么还要判断一次不太明白，毕竟前面CAS已经做了判断，此时应该没有其他线程可以操作，除非是扩容时有其他操作，咋们待会再看
                if ((tab = table) == null || tab.length == 0) {
                    // 注意此时sc >= 0,只有 sc = 0的时候才等于DEFAULT_CAPACITY 16，实际上当sc > 0时，代表此时数组是已经初始化过的，应该是不会进入initTable该方法了
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 如果 之前sc =0，则此时 n - 0.25n = 0.75n,也就是阈值 threshold，这种做法就是 取当前数组大小* 0.75
                    sc = n - (n >>> 2);
                }
            } finally {
                // sizeCtl记录了下一次调整数组大小的阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

<br/>

​        我们可以看到， ConcurrentHashMap中的数组初始化方法 initTable使用了 CAS来做了 乐观锁同步判断，判断的标志是 sizeCtl,如果现在还对 sizeCtl 迷糊的同学建议再往上重新看一下 sizeCtl的定义，

sizeCtl : volatile修饰，判断是否有线程在修改数组大小，为正数时，代表下一次的扩容的阈值



还记得put方法中当头节点的hash = MOVED即-1时，帮助转移吗？

接下来解析该方法

### helpTransfer 

开始之前，这里解释一下 ForwardingNode ，这是一个 静态内部类 static final class ForwardingNode<K,V> extends Node<K,V>

它HASH属性默认是 MOVED移动状态，也就是说这个节点默认是 MOVED移动状态的节点，可以用来做节点的状态比较判断,也可作为一个扩容扫描时下标跳跃的标志。

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    } 
    ...
 }
```

<br/>

还有 nextTable,它是下一次扩容时使用数组，扩容时不为null

```java
private transient volatile Node<K,V>[] nextTable;
```



```java
// 如果f节点在移动状态，则帮助转移
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        // resizeStamp 方法返回用于调整大小为n的表的标记位。向左移动RESIZE_STAMP_SHIFT时必须为负。
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
             // 如果 sizeCtl 无符号右移  16 不等于 rs （ sc前 16 位如果不等于标识符，则标识符变化了）
            // 或者 sizeCtl == rs + 1  （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16 位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
            // 或者 sizeCtl == rs + 65535  （如果达到最大帮助线程的数量，即 65535）
            // 或者转移下标正在调整 （扩容结束）
            // 结束循环，返回 table
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 如果以上都不是, 将 sizeCtl + 1, （表示增加了一个线程帮助其扩容）
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                //transfer 将每个bin中的节点移动和/或复制到新表中。,此时sc + 1，,SIZECTL就是2的15次方？？？这个数有什么用呢
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```



好，我们梳理一下helpTransfer的流程

1、判断当数组不为空，头节点处于移动标志时进行操作，不然返回原数组。

2、判断是否并发修改了，判断是否还在扩容。

如果还在扩容，判断标识符是否变化，判断扩容是否结束，判断是否达到最大线程数，判断扩容转移下标是否在调整（扩容结束），如果满足任意条件，结束循环。

如果不满足，并发转移原数组节点。



这个方法中的 sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1我都没怎么弄明白意义是什么，网上说是addCount方法中一个判断触发，但我未能理解。



### transfer

#### 源码解析

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 当CPU核数大于1时，stride 等于 桶的数量/8 再除 CPU核数，不然就一个CPU处理所有桶
    //  MIN_TRANSFER_STRIDE 默认16,该值至少应为 DEFAULT_CAPACITY。
    // 这里我们分析一下 当CPU为8核时，stride为0 < MIN_TRANSFER_STRIDE  所以stride = 16
    // 从现有服务器的核数分析 这么基本就是让 stride为16，为什么？  
    // 这里的目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少的话，默认一个 CPU（一个线程）处理 16 个桶
    // 所以我猜测这里是让每个线程处理16个下标对应的桶节点
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 当扩容数组为空时
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked") // 容量翻倍 n<<1
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // nextTable 是成员变量 、 更新 transferIndex 转移下标，就是原来tab的length
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    //  创建一个 fwd 节点，用于占位。当别的线程发现这个槽位中是 fwd 类型的节点，则跳过这个节点。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    //  首次推进为 true，如果等于 true，说明需要再次推进一个下标（i--），反之，如果是 false，那么就不能推进下标，需要将当前的下标处理完毕才能继续推进
    boolean advance = true;
    // 确定数组扩容结束标志
    boolean finishing = false; // to ensure sweep before committing nextTab
    //   死循环,i 表示下标，bound 表示当前线程可以处理的当前桶区间最小下标
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //  如果当前线程可以向后推进；这个循环就是控制 i 递减。同时，每个线程都会进入这里取得自己需要转移的桶的区间
        while (advance) {
            int nextIndex, nextBound;
              // 对 i 减一，判断是否大于等于 bound （正常情况下，如果大于 bound 不成立，说明该线程上次领取的任务已经完成了。那么，需要在下面继续领取任务）
            // 如果对 i 减一大于等于 bound（还需要继续做任务），或者完成了，修改推进状态为 false，不能推进了。任务成功后修改推进状态为 true。
            // 通常，第一次进入循环，i-- 这个判断会无法通过，从而走下面的 nextIndex 赋值操作（获取最新的转移下标）。
            //其余情况都是：如果可以推进，将 i 减一，然后修改成不可推进。如果 i 对应的桶处理成功了，改成可以推进。
             if (--i >= bound || finishing)
                advance = false;// 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进
            // 这里的目的是：1. 当一个线程进入时，会选取最新的转移下标。2. 当一个线程处理完自己的区间时，如果还有剩余区间的没有别的线程处理。再次获取区间。
            else if ((nextIndex = transferIndex) <= 0) {
                // 如果小于等于0，说明没有区间了 ，i 改成 -1，推进状态变成 false，不再推进，表示，扩容结束了，当前线程可以退出了
                // 这个 -1 会在下面的 if 块里判断，从而进入完成状态判断
                i = -1;
                advance = false;// 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进
            }// CAS 修改 transferIndex，即 length - 区间值，留下剩余的区间值供后面的线程使用
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;// 这个值就是当前线程可以处理的最小当前区间最小下标
                i = nextIndex - 1; // 初次对i 赋值，这个就是当前线程可以处理的当前区间的最大下标
                advance = false; // 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进，这样对导致漏掉某个桶。下面的 if (tabAt(tab, i) == f) 判断会出现这样的情况。
            }
        }
       // 注意，这里的n是原数组大小
       // 当i为负数，应该是代表扩容区间扫描完了
       // i >= n 代表到了 新数组的后半段区间
       // 这里 i + n >= nextn,nextn是新数组大小，从而推导出 i此时在后半段区间
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 当转移结束，sizeCtl为 2n * 0.75 即下一次扩容大小，原数组正式替换成新扩容后的数组
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }// 如果没完成扩容
          if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
              // 尝试将 sc -1. 表示这个线程结束帮助扩容了，将 sc 的低 16 位减一。 
              if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
              // 如果 sc - 2 不等于标识符左移 16 位。如果他们相等了，说明没有线程在帮助他们扩容了。也就是说，扩容结束了。 
              return;// 不相等，说明没结束，当前线程结束方法。 
              finishing = advance = true;// 如果相等，扩容结束了，更新 finising 变量 
              i = n; // 再次循环检查一下整张表 
          }

        } // 当数组对应下标节点为空时，填充为fwd占位标志，代表该节点不需要理会，可直接跳过，注意这里 f被赋值成下标首节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 如果状态为 MOVED，代表有其他线程正在帮助该下标转移数据，推进标志advance，代表可替换到下一个节点
        else if ((fh = f.hash) == MOVED) 
            advance = true; // already processed
        else {
            // 经过上述判断，确定了当前节点处于正常状态，加同步锁进行数据转移，f在上面判断赋值为当前下标的首节点
            synchronized (f) {
                 // 判断 i 下标处的桶节点是否和 f 相同
                if (tabAt(tab, i) == f) {
                     Node<K,V> ln, hn;// low, height 高位桶，低位桶
                    // 如果 f 的 hash 值大于 0,这代表是正常的链表节点 。TreeBin 的 hash 是 -2
                    if (fh >= 0) {
                        // 对老长度进行与运算（第一个操作数的的第n位于第二个操作数的第n位如果都是1，那么结果的第n为也为1，否则为0）
                         // 注意这里 n是2次幂，所以二进制编码中只有一个1，那么取于 length 只有 2 种结果，一种是 0，一种是1 
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        // 遍历链表节点
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        } // 当runBit为0，放置在低位桶中 ，这里如果看不明白，可以看一下我hashmap对于原桶中链表节点分散成两个链表的处理过程
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        } // 否则放置在高位桶
                        else {
                            hn = lastRun;
                            ln = null;
                        }}// 再次循环，生成两个链表，lastRun 作为停止条件，这样就是避免无谓的循环（lastRun 后面都是相同的取于结果）
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    } // 当桶中头节点是红黑树
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        // 这里的扩容和hashmap的链表resize扩容逻辑几乎一致 分配核心逻辑  hash & n
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }]
                        // 当下标的节点数小于等于 6,由红黑树转换为链表，不然创建一颗新的树
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

<br/>

#### 流程总结

1、判断CPU，这一步让每个CPU处理的数组中的桶数量保持一致，都为16

2、当扩容数组为空时，初始化扩容数组nextTable 

3、检测是否已经完成扩容，如果没有，开始循环遍历旧数组

4、当旧数组下标对应节点为空，填入fwd节点，标志可以跳过，如果旧数组下标为MOVED，代表已有线程正常转移该桶的节点，advance为true，进行其他下标判断，当当前下标节点处于正常状态时，开始加同步锁进行数据转移

5、当头节点hash大于0代表为链表节点，遍历所有节点，其中 做了 f.hash & n 判断，该结果只能为0或者1，将这些节点分配成两个链表节点中，一个链表放在新数组的 i坐标下，一个放在新数组的 i + n 坐标下。

6、 当头节点是红黑树时，同样分成两颗树，只是最终做了树的大小判断，当大小小于等于6，将其转换为链表

7、转移完当前下标i中的所有节点时，advance为true，代表可以扫描其他下标了。

8、当所有坐标都转移完成后，finish为true，标志tranfer结束 return



### 理解  f.hash & n 的含义

这里相信大家对 hash &  n (n为原数组大小) 为什么可以分成两个节点列表存储，并分配到 原坐标 i 和 i+n中会有疑问。

可以查看我在HashMap解析博客

扩容时涉及链表节点的下标分配的问题，旧数组的下标和新数组的下标有什么关联吗？

 这个需要从下面这个公式讲起。

[理解 hash & oldCap == 0的含义](http://www.eiletxie.cn/2019/12/08/聊聊HashMap中的设计/#理解-e-hash-amp-oldCap-0)



建议大家去查看一下背后逻辑

对这句话的总结

当 (e.hash & oldCap) == 0 时，证明节点Node 在 新数组下标 和旧数组的下标一致，我们用链表A保存

否则 证明 节点Node在 新数组的下标 = 旧数组下标 + oldCap一样，用链表B保存。



### 多线程帮助扩容

1. ConcurrentHashMap 支持并发扩容，实现方式是，将表拆分，让每个线程处理自己的区间。如下图：

 {% asset_img transfer.jpg  帮助转移 %}

这里触发的逻辑是 putval时检测当前下标节点的hash是否为 MOVED 即 -1，代表原数组正在扩容，那么当前线程会帮助其他下标节点的遍历转移。

并且多CPU下每个线程遍历处理的下标数量默认是16，如果是单CPU就是数组大小的桶数量

参考： https://juejin.im/post/5b00160151882565bd2582e0



### 个人建议

​    ConcurrentHashMap 分析起了真是要我老命了，我参考了几篇博客花了几晚才大致解析完核心方法的。看似 与HashMap实现逻辑差不多，但由于要高性能的并发安全，用了很多变量标志来进行状态判断,这使得有时候看某一段代码一头雾水。

​     这里如果有小伙伴也要分析，注意要理解 关键的状态标志的含义和 上面那就 hash & oldCap == 0的含义，理解 线程加锁是加在 线程下标下的头节点，每一次转移是当前下标下的节点转移就好了。

