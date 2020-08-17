---
title: 聊聊Semaphore的使用
date: 2020-06-06 19:59:43
tags:
- 并发
categories:
- JAVA
toc: true
---



​          对于Semaphore，我对其印象和使用都是比较模糊的，今天在lc中碰到了一道题目很适合用 Semaphore信号量来处理，同时在查看他人代码的过程中遇到了疑惑的地方，进行源代码的查阅和Debug后，对于Semaphore的使用和理解上面有了不少的提升，这里特意写一篇博客记录。



<!--more-->

### 事前说明：

源码的解析流程是比较长且繁琐的，建议大家可以根据这篇文章，打好相应的断点进行debug调试，调试的代码和具体的方法我在下文中会说明！



这里首先抛出几个问题。

1. Semaphore是什么？
2. Semaphore的适用场景？
3. Semaphore是如何实现的？

<br/>

## Semaphore是什么？

​            Semaphore是一个 计数信号量，类路径为 java.util.concurrent.Semaphore，你可以把Semaphore理解成 公司的共享雨伞（指定容量State），员工可以拿伞（获取资源acquire），可以归还伞（释放资源release），但是有一个前提 那就是 公司雨伞（容量state）是有限的。如果下雨天，有五百个员工，100把伞的话，当伞都被借走时，如果有员工想要借伞就必须要等别人归还伞，否则只能一直等待。

<br/>



## Semaphore的适用场景？

​     通过上面的描述，我想大家能明白Semaphore的常见用途。

​    1、 Semaphore可以用来 资源限流。例如： 大量人员购票，车站售票窗口有限。 又或者是具体应用场景 数据库连接池中同时连接的线程不能超过指定的连接数量

​    2、Semaphore可以用来进行状态切换，例如： 红绿灯、开关，但是要理解Semaphore其实只适合少量的状态变化，你看完下文理解了就明白为啥不适合大量的状态切换了。

<br/>

​        这里说明： 我之所以Semaphore学的比较模糊是因为我没有在具体的场景中将其合理的运用过，这导致我对于Semaphore的理解很浅，semaphore进行资源限流，我想大家都应该比较了解，无非就是指定 Semaphore容量，进行acquire和release的过程。

### 资源限流的场景

例如：

春节人民取票，火车站只有6个售票窗口

```java
Semaphore windows = new Semaphore(6);
// 春节无数人进行取票
while(true){ // 这里图省事就直接无限循环的，现实中往往是一个 for(0 -> n) 的循环
    windows.acquire(); // 占用一个窗口取票，没有窗口就会堵塞等待
    .... 执行取票逻辑
    windows.release(); // 离开该窗口
}
```

但是如何灵活的使用 Semaphore进行状态切换，执行每个状态下该有的逻辑，我想大家应该是比较懵的，这里推荐一道应用场景题，从该题目来入手

<br/>

### 状态切换的场景

[**1115. 交替打印FooBar**](https://leetcode-cn.com/problems/print-foobar-alternately/)

两个不同的线程将会共用一个 FooBar 实例。其中一个线程将会调用 foo() 方法打印 foo，另一个线程将会调用 bar() 方法打印 bar。

请设计修改程序，以确保 "foobar" 被输出 n 次。

```java
class FooBar {
    private int n;

    public FooBar(int n) {
        this.n = n;
    }

    public void foo(Runnable printFoo) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            
            // printFoo.run() 打印 foo
            printFoo.run();
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            
            // printBar.run() 打印 bar
            printBar.run();
        }
    }
}
```



当然，这个题目可以用 Lock来做、也可以用Semaphore，甚至可以用 volatile变量进行自旋锁判断，只是性能较差，不推荐。

请大家思考一下，用 semaphore怎么做？

 {% asset_img think.jpg  思考表情图 %}

这里我们先来公布一下答案：

```java
class FooBar {
   private int n;

    public FooBar(int n) {
        this.n = n;
    }

    Semaphore foo = new Semaphore(1);
    Semaphore bar = new Semaphore(0);

    public void foo(Runnable printFoo) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            foo.acquire();
            printFoo.run();
            bar.release();
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            bar.acquire();
            printBar.run();
            foo.release();
        }
    }
    // 测试代码
    public static void main(String[] args) {
            FooBar fb = new FooBar(2);
            new Thread(() -> {
                try {
                    fb.foo(new Runnable() {
                        @Override
                        public void run() {
                            System.out.print("foo");
                        }
                    });
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        
            new Thread(() -> {
                try {
                    fb.bar(new Runnable() {
                        @Override
                        public void run() {
                            System.out.print("bar");
                        }
                    });
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
}
```

<br/>

这里我想大家对这段代码会有疑问 

```java
public void foo(Runnable printFoo) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            foo.acquire();
            printFoo.run();
            bar.release();
        }
    }
```

为啥，bar.release(); 可以这么玩？bar的信号量不是为0吗？还能接着 release()？  Semaphore bar = new Semaphore(0); 并且在 bar方法中还可以获取信号量？

 {% asset_img what.jpg  不可思议图 %}

<br/>

#### release()方法分析

别急! 这个时候得看源码分析一下操作了。

```Java
public class Semaphore implements java.io.Serializable {
    public void release() {
        sync.releaseShared(1);
    }
    private final Sync sync;
    // 使用AQS实现的同步器,公平版（指锁竞争机制公平，排队先到先得)
    abstract static class Sync extends AbstractQueuedSynchronizer {
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
    }
    //  NonFair version 非公平版
    static final class NonfairSync extends Sync {}
}
```

我们可以看到 Semaphore的release方法调用的 是  AQS类的releaseShared(1);

AbstractQueuedSynchronizer.releaseShared()方法

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

重头戏来了，我们来重点分析这两个方法！！

**分析步骤1：  if (tryReleaseShared(1))**

**分析步骤2：   doReleaseShared();**

tryReleaseShared(int arg)方法，由于子类中有实现，会实现子类的方法，注意看上面Semaphore 对应的 Sync内部静态类中的该方法（这里强烈建议自己debug一下）

注意： 我们讨论的是 下面这代码为什么信号量为0了，还可以直接释放信号量，并且还可以获得信号量的操作。

```java
 Semaphore bar = new Semaphore(0);
 bar.release();
 bar.acquire();
```

<br/>

接着看Semaphore 源码 tryReleaseShared(1)

```java
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                //  getState()为0，尝试释放的releases为1
                int current = getState();
                int next = current + releases;
                // (next = 1) > (current = 0)
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                    // 注意这里， 这里直接用CAS 把 current设置为 next，也就是把信号量从0变成了1！！！
                    // 无限循环，自旋CAS来确认增加了当前信号量 = 原来的信号量 current  + 要释放的信号量 releases
                if (compareAndSetState(current, next))
                    return true;
            }
 }
```

由于  if (tryReleaseShared(1)) 返回的是true，信号量++了，此时再看 doReleaseShared();

这里 doReleaseShared() 方法是 AQS类中的，子类 Semaphore中Sync 同步器并未实现该方法

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

这一段代码，如果没研究过 AQS的话会看得很迷糊，我这里为了篇幅起见就简短的解释一下。

首先大部分我们使用的同步类都是通过 继承 AQS来实现的。

因为AQS 将 线程安全中大部分的逻辑都已经封装好了， 继承的子类只需要调用它的方法来做 信号量的修改，以及决定是 共享锁的方式还是 独占锁的方式来表现子类的特性就行。



AQS中有个 CLH 等待队列,结构是一个 双向链表，它的用途就是 当一个资源被A线程占有时，其他申请资源的线程都会进入 CLH等待队列中排队等待 A线程释放资源。

**上面 doReleaseShared方法，就是释放资源信号量后，使得CLH队列 head节点资源，并且将head节点 = head.next的这样一个过程。**





这样一来， bar.acquire();就能够说得通了,因为 bar.release()方法 是将原本 为零的信号量， 通过CAS 变成了 state = 1



####  **release()方法总结**

Semaphore的 release()方法，不会考虑原本的信号量的数量问题，而是通过CAS实现 当前信号量加一的操作，从而实现了信号量从无到有的这个过程。

<br/>

#### acquire() 方法分析

那 acquire() 方法呢,这个肯定是会要求当前信号量大于0的，否则就会堵塞请求的线程，为什么呢？

因为不这样，那还叫啥 线程安全的同步器呀，都没有堵塞机制了！！

不信，咋们看代码

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

调用了 AQS类的acquireSharedInterruptibly方法

```java
 public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
          // 判断线程是否中断，已经中断的话直接抛异常
        if (Thread.interrupted())
            throw new InterruptedException();
           
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

由于子类对应的 Semaphore的同步器 实现了  tryAcquireShared(arg)方法，必然会调用到具体实现子类 Semaphore 的静态内部类 非公平同步类 NonfairSync

```java
static final class NonfairSync extends Sync {
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

我们可以查看 Semaphore 的静态内部类 Sync实现的 nonfairTryAcquireShared(1)方法

```java
public class Semaphore implements java.io.Serializable {
    abstract static class Sync extends AbstractQueuedSynchronizer {
         final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                // 假设此时信号量getState()为0
                int available = getState();
                // 那么 remaining = -1
                int remaining = available - acquires;
                // 注意： 如果此时 信号量大于0，那么这里会直接通过 CAS 设置 信号量-1，并且返回最新的信号量值，这就意味着获取成功了
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
}
```

由于if (tryAcquireShared(arg) < 0)的条件中 tryAcquireShared(1) = -1，所以执行if下的逻辑，调用   doAcquireSharedInterruptibly(arg);

```java
 if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
```

AQS类中的doAcquireSharedInterruptibly方法

注意： 此时是获取

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
        // 在CLH等待队列中添加一个节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            // node.predecessor() ：返回node的上一个节点
            final Node p = node.predecessor();
            if (p == head) {
                // 当上一个节点是头节点是，再次调用tryAcquireShared(arg)方法，这个方法是 Semaphore的内部类实现的，具体看下面
                // 当资源还没有被release释放时， 即信号量还是为0时，tryAcquireShared(arg)方法肯定返回的还是 -1
                int r = tryAcquireShared(arg);
                // 如果 getState() = 0 ,那么 r为-1，接着循环执行，直到其他线程释放信号量了，那么 tryAcquireShared(arg)成功，r >= 0了
                if (r >= 0) {
                    // 信号量大于0了，这就意味着头节点p可以获取信号量并移除该头节点，那么设置当前 线程节点为 CLH队列的头节点，然后跳出循环
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

这里贴上添加等待节点的方法addWaiter来让大家了解 CLH等待队列

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

Semaphore也可以定义 公平锁与非公平锁，默认是 非公平锁，我们这里为了篇幅起见，只介绍非公平锁，因为逻辑基本差不多

```java
static final class NonfairSync extends Sync {
    protected int tryAcquireShared(int acquires) {
        // 由于是非公平锁，所以会直接取抢信号量，不需要排队
        // 再次执行 nonfairTryAcquireShared(acquires)，假设此时资源还没释放，信号量为零，那么这里肯定返回的是 -1，具体逻辑上面代码已经分析了
        return nonfairTryAcquireShared(acquires);
    }
}

static final class FairSync extends Sync {
    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

<br/>

#### acquire方法总结

1、 当执行 tryAquire方法时，它会先通过 sync.nonfairTryAcquireShared(1)  >= 0 来判断，当前信号量是否大于0

2、如果大于0，则nonfairTryAcquireShared()方法中，直接通过CAS将 信号量减一并返回true

3、如果小于0，先将当前线程封装成一个 node节点添加到 CLH等待队列中，然后不停的循环判断，是否node的上一个节点是否为头节点

4、循环判断，当node.prev = head时，通过 tryAcquireShared(arg)方法判断 信号量是否大于0，大于0，将node设置为 CLH队列的头节点

<br/>

这里细心的同学会有疑问，虽然说当信号量为0时，线程acquire无限等待CLH队列的头节点的出列，直到这个被封装的线程等待节点 waiterNode成为了CLH等待队列的头节点，但是！！

它终究没有获取到信号量啊！！！怎么就跳出方法了呀，这就意味着还没获取资源就跳出循环，这不没有起到堵塞的效果吗？？？

 {% asset_img success.jpg  很好你成功了图 %}

good！不要急，唯一有机会继续搞幺儿子的就是   setHeadAndPropagate(node, r);方法了

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
   // 因为node为新的头节点了，那么 h肯定为null了，因为已经旧的头节点被释放了
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 这里忽略if,正常情况都会执行下面的语句
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
// 这个nextWaiter和SHARED，就很复杂了，留到下一篇文章分析AQS再说吧
final boolean isShared() {
    return nextWaiter == SHARED;
}
```

 doReleaseShared();方法会无限循环判断，直到头节点改变了，证明 node已经获取到信号量了，此时退出循环，正式结束整个acquire方法的逻辑！！

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 这里对于节点状态的CAS替换判断，涉及CAS，留到下一篇
            // 但是大家应该可以通过官方的注释和无限循环来理解doReleaseShared的作用
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

<br/>

#### acquire方法再次总结

1、 当执行 tryAquire方法时，它会先通过 sync.nonfairTryAcquireShared(1)  >= 0 来判断，当前信号量是否大于0

2、如果大于0，则nonfairTryAcquireShared()方法中，直接通过CAS将 信号量减一并返回true

3、如果小于0，先将当前线程封装成一个 node节点添加到 CLH等待队列中，然后不停的循环判断，是否node的上一个节点是否为头节点

4、循环判断，当node.prev = head时，通过 tryAcquireShared(arg)方法判断 信号量是否大于0，大于0，将node设置为 CLH队列的头节点

5、在设置node为CLH队列的头节点的这个方法中doReleaseShared会无限循环判断当前头节点node是否已经改变了，改变了说明已经获取到信号量了，acquire方法结束！