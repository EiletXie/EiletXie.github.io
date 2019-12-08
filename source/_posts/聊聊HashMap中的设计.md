---
title: 聊聊HashMap中的设计
date: 2019-12-08 15:06:47
tags:
- 集合
toc: true
---



## 聊聊HashMap中的设计

​             刚开始了解HashMap的时候，知道它底层是 数组 + 链表 + 红黑树的数据结构。最近重新看源码做复习的时候，发现很多写法我可以借鉴，例如变量判断中赋值，取比某个整数大的最小2次幂等等。



所以我这里主要介绍以下几个方面的内容。

1、 HashMap 的底层组成原理，为什么这样设计？

2、HashMap 扩容逻辑和PUT SET方法

3、 如何取一个不小于给定整数的最小二次幂



<!--more--> <br/>

## HashMap初步介绍：

HashMap 位于java.util包下。继承关系如下

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

基于哈希表的实现了Map`接口，并允许`null的`值和`null键。 

<br/>

我们需要了解两个重要的参数

**initialCapacity** 初始容量 和 **loadFactor** 负载因子

```java
// new HashMap() 时默认初始容量 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
// 负载因子 默认为0.75 当数组中的个数 > 数组大小 * 负载因子 时就会扩容 ,这个 阀门阈值 为 loadFatoar
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```



可以看出我们初始化时 HashMap<K,V> map  = new HashMap<>()

```
 public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; 
    }
```

​         像我们平常使用的就是这种方式，这种方式实际上是不好的，如果在业务场景中hashmap中要插入大量的key-value的话，需要不断的从初始容量扩容多次，这里的原理我聊到  put方法的时候再解释。

<br/>

## HashMap的底层组成

​         HashMap在 JDK 1.7 之前 是 数组 + 链表的组成结构，JDK1.8开始 转变为 数组 + 链表 + 红黑树。

​        这里我列出部分核心的结构代码。

```java
 // 数组 table 存储了链表头结点
transient Node<K,V>[] table;

//  链表节点 Node ,单向链表，保留节点的hash和 key-value
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
 }

// 红黑树 TreeNode, 注意 这里 TreeNode 和 Node 都继承了 Map接口
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
}
```

底层组成图:

{% asset_img 底层组成.jpg 底层组成 %}

<br/>

### 为什么要选择这种底层组成呢？

​         首先为什么初始选择链表而不是AVL树?

​         这是内存占用与存储桶内查找复杂性之间的权衡。请记住，大多数哈希函数将产生非常少的冲突，因此为大小为3或4的桶维护树将是非常昂贵的，没有充分的理由。

​      作为参考，这是一个HashMap的Java 8 impl（它实际上有一个很好的解释，整个事情如何工作，以及为什么他们选择8和6，作为“TREEIFY”和“UNTREEIFY”阈值）

​     其实这里还涉及一个 泊松分布的知识点，感兴趣的同学可以自行了解一下。        

参考： https://stackoverflow.com/questions/42422469/why-does-a-hashmap-contain-a-linkedlist-instead-of-an-avl-tree

<br/>

当链表长度达到8时，转换的是红黑树

#### 为什么选择红黑树而不是 AVL树?

RB-Tree和AVL树作为BBST (balance binary search tree 自平衡二叉搜索树)，其实现的算法时间复杂度相同，AVL作为最先提出的BBST，貌似RB-tree实现的功能都可以用AVL树是代替，那么为什么还需要引入RB-Tree呢？

红黑树不追求"完全平衡"，即不像AVL那样要求节点的 `|balFact| <= 1`，它只要求部分达到平衡，但是提出了为节点增加颜色，红黑是用非严格的平衡来换取增删节点时候旋转次数的降低，任何不平衡都会在三次旋转之内解决，而AVL是严格平衡树，因此在增加或者删除节点的时候，根据不同情况，旋转的次数比红黑树要多。

1. 就插入节点导致树失衡的情况，AVL和RB-Tree都是最多两次树旋转来实现复衡rebalance，旋转的量级是O(1)
    删除节点导致失衡，AVL需要维护从被删除节点到根节点root这条路径上所有节点的平衡，旋转的量级为O(logN)，而RB-Tree最多只需要旋转3次实现复衡，只需O(1)，所以说RB-Tree删除节点的rebalance的效率更高，开销更小！

2. AVL的结构相较于RB-Tree更为平衡，插入和删除引起失衡，如2所述，RB-Tree复衡效率更高；当然，由于AVL高度平衡，因此AVL的Search效率更高啦。

3. 针对插入和删除节点导致失衡后的rebalance操作，红黑树能够提供一个比较"便宜"的解决方案，降低开销，是对search，insert ，以及delete效率的折衷，总体来说，RB-Tree的统计性能高于AVL.

4. 故引入RB-Tree是**功能、性能、空间开销的折中结果**。
    5.1 AVL更平衡，结构上更加直观，时间效能针对读取而言更高；维护稍慢，空间开销较大。
    5.2  红黑树，读取略逊于AVL，维护强于AVL，空间开销与AVL类似，内容极多时略优于AVL，维护优于AVL。
    基本上主要的几种平衡树看来，**红黑树有着良好的稳定性和完整的功能，性能表现也很不错，综合实力强**，在诸如STL的场景中需要稳定表现。

   

{% asset_img RB和AVL比较.jpg  RB和AVL比较%}

参考: https://www.jianshu.com/p/37436ed14cc6

<br/>

## HashMap的PUT方法

put方法调用的是putVal 方法，我们分析一下源码

这里要注意，变量在判断的时候都被赋值了

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //1、 若 HashMap 未被初始化，则进行初始化操作。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //2、 当对应数组下标为空时，填入节点 hash重新优化使得范围在 0~n-1 (上面判断使得 n = tab.length） 
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 当对应数组下标有链表节点时
        Node<K,V> e; K k;
        // 在第二步判断中，p 被赋值为 hash后对应的下标首节点，这里当首节点与插入的节点的 hash 和 key都一致时，首节点赋值给插入节点 e = p
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // 当 首节点为红黑树时，检测返回一个新建的树节点赋值给待插入的节点e
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 否则 此时朝着链表向下遍历，在尾节点创建新节点 传入要插入的相关值
            for (int binCount = 0; ; ++binCount) {
                // 当遍历到链表的尾节点时，注意每次判断都在重新赋值 e = p.next
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 判断当前节点的链表长度是否大于 TREEIFY_THRESHOLD 树的临界值 8,大于时进行是否转为红黑树判断
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                    //  treeifyBin 会判断tab总节点数如果小于 MIN_TREEIFY_CAPACITY 64时，重新排列resize节点，大于时则将该数组下标的链表转换成红黑树
                        treeifyBin(tab, hash);
                    break;
                }
                // 这里是判断链表节点是否有一样的 key 和hash值，如果有无需重复插入，直接跳出
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 判断 待插入的节点e 存在于 map时，替换 oldValue (上面的遍历中 如果e的key不存在则创建了节点，存在则指向已知节点)
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            // 如果是已存在的节点，替换值后直接返回了被替换的新值
            return oldValue;
        }
    }
    ++modCount;
    // 值得注意的是 到这里来的时候，证明 插入的是新的节点，都会size++进行 一次 threshold 阈值（扩容临界值）的判断
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

<br/>

### PUT方法的流程图：

{% asset_img put.jpg  put%}

<br/>

## HashMap的GET方法

GET 方法相对而言要简单不少，毕竟不涉及数据的存储结构的变化。

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 判断 数组是否为空或 对应的下标有无头节点，否则证明无对应key返回null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
            // 检查头节点是否和给定节点匹配
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
             // 当头结点为红黑树类型，进行树的查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
             // 否则为 链表节点，直接遍历后续节点直到为空即可
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

<br/>

## HashMap的扩容逻辑

​       当hashmap未赋值时，或者节点达到一定数量时都会在put方法中出发resize扩容方法。我们来分析一下其源码

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //  注意 这两个参数 oldCap 旧数组的容量大小  oldThr 旧的扩容阀值 = 负载因子 * 旧数组容量大小
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //1、 设定 newCap 新数组大小为 旧数组大小的两倍 （这里发现 当新容量大于 2的30次方时，这里未做处理 在下方的 2步骤进行重新赋值newThr）
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // 初始容量置于阈值
        newCap = oldThr;
    else {              
        // 旧数组的容量和 扩容的阈值都为0  代表是 new HashMap() 这种情况开始赋值给newCap和newThr,可以回看 put方法 判断tab是否为空时调用的就是 resize给tab赋值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    if (newThr == 0) {
        // 2、 发生newThr为空, 可能是 newCap大于限定最大值 MAXIMUM_CAPACITY ,这里赋值 newThr为  Integer.MAX_VALUE
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 遍历 旧数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 当旧数组对应下标的头节点不为空时
            if ((e = oldTab[j]) != null) {
                // 将 oldTab[j]置为 null，加快JVM回收
                oldTab[j] = null;
                // 如果头节点无下一节点时 直接重新 计算下标  e.hash & (newCap - 1) 进行赋值
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 注意 这时候的节点 为红黑树 并且 e.next不为空时，我们拆分头节点，并进行自平衡，由于篇幅关系，刚兴趣的同志可以看看split方法中红黑树的remove节点逻辑
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                // 只剩最后一种情况了，是链表，并且 e.next不为空 这里涉及一个 下标重排的巧妙优化，我会在下面讲解
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 这句话是关键，下面会详细解释这个公式
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

<br/>

扩容时涉及链表节点的下标分配的问题，旧数组的下标和新数组的下标有什么关联吗？ 这个需要从下面这个公式讲起。

<br/>

### 理解 (e.hash & oldCap) == 0

在讲解这个公式之前，我们需要重温下面这几个知识点

- 旧数组的容量大小是 2的 m次方, 这里我假设是 4次方 ，16大小，后面会以这个大小示例
- 新数组是旧数组的容量大小的两倍
- put方法中 e.hash & (n-1) 实际上求的是 hash的后m位,即对于的旧数组下标

<br/>

值得注意的是，我们可以确定 在同一个旧数组下标下的所有节点，它们的hash后m位是一致（数组容量是 2的m次方） 这里 方便解释指定 m=4。

为什么？ 如果不一致 那么 e.hash & (n-1) 就不可能在同一个下标下。

这里假设它们在该下标下的 hash后4位为 xxxx

这里我们可以看下图推演

{% asset_img hash.jpg  hash%}

<br/>

那在旧数组的指定下标下的这些节点在 新数组下对应的下标计算就是 

e.hash & (2n - 1)  

它与 旧数组下标计算 hash & (n - 1) 有什么区别呢？

它多保留了一位，它是保留了节点hash值后5位了，所以出现了两种情况,这里hash长度总共32位，我简写只保留后8位，前面都因为在 & 数组容量 操作中都为0了

```java
0000 xxxx
0001 xxxx
```

注意 二进制 x不是为0,就是为1



所以发现没有，旧数组某个下标下的所有节点Node 在分配到新数组，重新计算下标时,出现了两种情况。

一种是 0000 xxxx 这种情况 和旧数组的下标  xxxx是一致的，我们用链表A保存起来

另外一种 0001 xxxx 这种情况我们需要推演一下

注意： 我们这里假设旧数组的容量是 2的4次方 16

1 xxxx = xxxx + 10000 = 旧数组下标 + 旧数组容量 oldCap

所以第二种情况 在新数组的下标 = 旧数组的下标 + 旧数组的容量 oldCap

<br/>

那我们如何判断在 旧数组某下标下的所有节点， 它们是属于哪种情况呢？

我们只需要计算 倒数第5位是 0,还是1不就行了吗？

 hash & 1 0000 =>  hash & oldCap == 0 ?



所以现在我们能理解这个公式了吗？

**(e.hash & oldCap) == 0**

<br/>

### 总结

当 (e.hash & oldCap) == 0 时，证明节点Node 在 新数组下标 和旧数组的下标一致，我们用链表A保存

否则 证明 节点Node在 新数组的下标 = 旧数组下标 + oldCap一样。



所以现在我们可以很轻松的明白 resize方法中的最后一个判断了吧。

不得不说 写这段代码的大佬太牛了，短短一句话包含这么多逻辑。



<br/>

## HashMap如何取不小于给定整数的最小二次幂

   这就要聊聊 HashMap中的 **tableSizeFor** 方法了

```java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

该方法做到了，返回一个不小于给定整数的最小二次幂

<br/>

先来分析有关n位操作部分：先来假设n的二进制为01xxx...xxx。接着

对n右移1位：001xx...xxx，再位或 | ：011xx...xxx

对n右移2为：00011...xxx，再位或 | ：01111...xxx

此时前面已经有四个1了，再右移4位且位或可得8个1

同理，有8个1，右移8位肯定会让后八位也为1。

**综上可得，该算法让最高位的1后面的位全变为1。**

<br/>

最后再让结果n+1，即得到了2的整数次幂的值了。

现在回来看看第一条语句：

int n = cap - 1; 

　　让cap-1再赋值给n的目的是另找到的目标值大于或等于原值。例如二进制1000，十进制数值为8。如果不对它减1而直接操作，将得到答案10000，即16。显然不是结果。减1后二进制为111，再进行操作则会得到原来的数值1000，即8。

<br/>

## 个人体会：

​         HashMap等Java源码在判断和数组扩容等场景下，大量使用了位运算来保证代码整洁和执行的高效性，我们在操作数组的部分业务场景中完全可以学习这种思路，包括判断中赋值，取得最小二次幂等等，还是得多刷点位运算的算法题来提升自己这种简洁代码的能力。

