---
title: AQS使用及分析之AQS分析
date: 2019-11-20 16:01:29
tags:
  - Java
  - AQS
categories:
  - Java
  - Concurrent
---

# AQS使用及分析之AQS分析

接下来的这篇文章将详细分析AQS，AQS全称是AbstractQueuedSynchronizer，翻译过来是抽象队列同步器，也即AQS是一个基于队列实现的抽象同步器

AQS封装了获取锁，释放锁等操作，同时暴露一些try开头的方法交给子类来实现，通过继承AQS可以非常方便地实现一个线程安全的并发同步器，在`java.util.concurrent`包中的大多数高层次同步器，如`CountdownLatch`、`ReentrantLock`等都是通过AQS实现的

常用的基于AQS高层次同步器有如下几个

- CountdownLatch
- CyclicBarrier
- Semaphore
- ReentrantLock

这篇文章只分析AQS提供的通用逻辑实现，分析完AQS之后，后面再用新的篇章分析基于AQS的同步工具的具体实现细节

> 本文参考自[javadoop](<https://javadoop.com/>)的
>
> [一行一行源码分析清楚AbstractQueuedSynchronizer](<https://javadoop.com/post/AbstractQueuedSynchronizer>)
>
> [一行一行源码分析清楚 AbstractQueuedSynchronizer (二)](<https://javadoop.com/post/AbstractQueuedSynchronizer-2>)
>
> 非常感谢大神无私地分享

<!--more-->

## 核心概念

在具体分析AQS之前，有几个核心概念需要先弄清楚，这些概念是看懂接下来内容必不可少的

### 链表

链表是一个基础的线性数据结构，节点之前通过"指针"链接起来，从而构成一个完整的链，`java.util.LinkedList`就是一个非常典型的链表的实现

链表根据实现细节的不同，有三种比较典型的链表

- 单向链表，节点通过单向指针链接起来，每个节点只能访问其后面的节点
- 双向链表，每个节点通过前向指针，后向指针分别链接前节点、后节点，每个节点可以访问其前后节点
- 循环链表，链表收尾相连，构成一个环，有两种典型实现
    - 单向循环链表
    - 双向循环链表

AQS中获取不到锁的排队就是通过一个双向链表来实现，该队列也称为等待队列(同步队列)

同时也通过内部类`ConditionObject`，来实现`Condition`的语义，在某个条件上等待的语义就是通过一个单向链表来实现，该队列也称为条件队列

### 锁

锁，这个概念并不陌生，在并发环境下，对于临界区的访问，为了避免并发操作对数据带来不一致性(也称为竞态条件)，通常需要先获取锁再执行对应操作

根据同时可以被持有对象的不同，锁可以分为两种

- 独占锁，也称为X锁，顾名思义，同时只能被一个线程持有，只有加锁的线程才能解开锁
- 共享锁，也称为S锁，允许同时多个线程持有(同时加锁或者解锁都是可以)

根据是否可被中断，锁可以分为两种

- 可中断锁，顾名思义，收到中断信号，并且会处理中断信号，在Java中通常表现为抛出中断异常`InterruptedException`
- 不可中断锁，发生中断异常时，仅记录中断信号而不会抛出中断异常

根据获取锁的时机不同，锁可以分为两种

- 抢占锁，也称为非公平锁，释放锁的那一刻，允许新来的线程参与锁的抢占
- 不可抢占锁，也称为公平锁，按照先来后到的顺序分配锁

根据锁的策略，可以分为两种

- 乐观锁，认为竞争本身很小，所以大部分情况下是不需要获取锁来访问临界区
- 悲观锁，认为竞争大，每次访问临界区都需要获取锁

上面的几种分类是常见的锁的分类，这些分类之间是可以相互组合的，如不可中断独占锁，可中断共享锁等

在AQS中，提供了独占锁和共享锁的实现，同时，提供一个方法用于实现公平锁、非公平锁

### 无锁化

在旧版的JDK中，提供了基于监视器的`synchronized`关键字，用于作为锁的实现，该锁是自动获取以及释放，有时候也称为对象锁，因为锁的目标就是某个对象

使用锁的好处的使用方便，直观，但也有比较明显的缺点

1. 通常获取锁的时候，如果无法获取到锁，则线程本身会被阻塞，等到锁重新释放的时候，才会重新尝试获取锁，当并发比较大的时候，对于锁的竞争会非常激烈，容易影响性能
2. 在一些场景中，临界区本身非常小，对于临界区的操作时间比获取锁、释放锁两个操作加起来的时间还小，也就容易造成锁本身带来的消耗大于实际工作的时间

于是，无锁化的概念逐渐提出，无锁化，顾名思义，对临界区的操作本身不再通过加锁的形式来访问，有时候，这种形式也称为乐观锁，目前采用得比较多的就是通过CAS来实现无锁化操作，CAS本身需要CPU提供支持，关于Java中的CAS实现，可以大致参考下之前的文章：[AtomicXXX使用及分析](<https://www.xuhuanfeng.cn/2019/10/06/AtomicXXX%E4%BD%BF%E7%94%A8%E5%8F%8A%E5%88%86%E6%9E%90/#more>)

在AQS中，对于临界区(锁状态、链表节点改变等)的操作，都是通过CAS来实现的

### 条件等待

条件等待在编程中非常常见，某一个线程需要某些资源，而这些资源目前还没有准备完成，此时线程就需要等待该资源准备完成之后再运行，在等待期间，线程需要挂起，这个过程也称为"条件等待"。

等待某个条件发生，如常见的生产者消费者模型中，当缓冲区满了的时候，生产者需要等待缓冲区有空间，当缓冲区空的时候，消费者需要等待缓冲区有内容

在Java中，每一个对象都可以作为一个条件，多个线程可以在该对象上进行等待，如常见的`obj.wait()`，在进行条件等待的时候，需要先获取该对象锁，否则会抛出监视器异常，执行wait操作之后，线程会被挂起，同时，线程持有的锁会被释放；当被其他线程唤醒，即`notify()`的时候，线程会被唤醒，重新获取锁，然后继续执行(注意，此时条件不一定满足，因为从条件发生到线程获取到CPU执行存在时间差，可能线程继续执行的时候，条件又不满足了，所以一般是将条件等待放在循环中，唤醒之后重新检查一下是否条件满足)

## AQS核心内容

### Node结构

Node是AQS的一个内部类，该类本身所包含的属性不多，却是整个AQS的核心，有如下几个属性

- waitStatus，等待状态
- thread，节点对应的线程
- next，指向**等待队列下一个节点**
- prev，指向**等待队列前一个节点**
- nextWaiter，指向**条件队列下一个节点**

其中的waitStatus是比较复杂的，用于标志当前节点所处的状态，有以下几个值

- CANCELLED：1，表示线程取消等待
- 0，默认值
- SIGNAL：-1，表示**下一个节点**需要被唤醒
- CONDITION：-2，表示当前节点处于等待条件发生
- PROPAGATE：-3，表示下一个节点等待共享锁的线程需要被无条件通知



nextWaiter是另外一个比较有趣的字段，前面提到了，AQS支持条件等待，当多个线程在同一个条件上等待时，显然，也是需要将其合理的组织起来

Node中通过类型的Node的nextWaiter将其串联起来，其值有两种

- 当处于共享模式时，节点的值均为`SHARE`
- 当处于排他模式时，为等待同样条件的节点(默认为null，表示没有)



当节点处于等待获取锁时，等待队列如下图所示

![等待队列.png](http://ww1.sinaimg.cn/large/b162e9f4gy1g8xfdqkx98j20zz0a1405.jpg)

当节点处于等待某个条件时，条件队列如下所示

![条件队列.png](http://ww1.sinaimg.cn/large/b162e9f4gy1g8xf8mxafmj21050abjt2.jpg)

### AQS结构

AQS的实现细节非常复杂，各种情况错综复杂，考虑地非常详细，但是，AQS的结构本身非常简单，由一个head一个tail构成等待队列，一个state代表锁状态，一个exclusiveOwnerThread代表独占锁持有线程

state就是一个整数，0表示当前锁没有被占有，大于0表示重入，为了保证可见性，同时被声明为`volatile`

exclusiveOwnerThread代表的是当前占据独占锁的线程，该属性继承于AbstractOwnableSynchronizer

AQS的完整结构如下

![AQS.png](http://ww1.sinaimg.cn/large/b162e9f4gy1g8xivaory3j20vv0j441s.jpg)

### 获取锁

前面提到了，AQS提供的核心功能其实就是获取锁以及释放锁这两个操作，对于AQS的理解，关键也在于这两个操作，根据锁类型的不同，AQS提供了独占锁与共享锁两种不同类型的操作，接下来详细分析每一个操作

#### 获取独占锁

AQS中，获取锁的操作通过acquire开头的一些方法完成的，根据是否可被中断，分为两种：可中断以及不可中断，关于这两者前面已经介绍了，这里就不展开，直接分析AQS的实现

##### 不可中断

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`acquire`方法是一个模板方法，留下了`tryAcquire`交由子类实现，也即子类根据实现的不同，决定什么时候能够获取成功(返回true)，什么时候获取失败(返回false)

由于获取成不成功是由子类决定的，并且不同的工具的含义也不同，这里我们只重点看下获取失败的操作

根据独占锁的语义，一把锁同时只能被一个线程持有，在持有锁期间，其他线程尝试获取锁，都应该被阻塞或者挂起，结合上面的前面等待队列的图示，不难猜出，获取失败之后的大致流程就是将当前线程包装成链表节点，然后添加到链表中，最后挂起自己，等待唤醒

> addWaiter操作

首先是addWaiter操作，在AQS中，等待队列的一个节点也称为一个waiter，那么，这个方法就很显然了，将线程包装成一个waiter，然后添加到等待队列中

```java
// 参数mode代表的含义是当前线程的等待类型，这里是独占锁，那就是EXCLUSIVE啦
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    // 初始情况下，head=tail=null
    
    // pred != null说明等待队列初始化完成了
    // 那就尝试将自己挂在队列尾部，成为新的尾部
    if (pred != null) {
        // 将当前节点的prev指针指向原来的tail(这一步线程安全)
        node.prev = pred; // 1
        // 通过cas将原来的tail设置为自己
        // 注意，这里是存在并发情况的，如果有两个线程同时将自己设置为尾部
        // 那么存在一个被另一个覆盖的情况，所以需要通过CAS来保证只有一个成功
        if (compareAndSetTail(pred, node)) { // 2
            // 如果cas成功，则将原尾部的next指向自己，使自己成为新的尾部
            pred.next = node; // 3
            return node;
        }
        
        // 上面的1、2、3三个步骤不能变换顺序，否则会出现数据错乱
    }
    
    // 如果到了这里，有两种情况
    // 1、队列未初始化
    // 2、CAS失败，表明同一时刻有另一个线程抢先成为尾节点了
    
    // 那应该怎么办呢？
    // 很显然，入队是必须的，那就只能一直尝试入队，直到成功为止了
    enq(node);
    return node;
}
```

> enq操作

```java
// 该操作意图很简单，一直尝试入队，直到成功为止
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // 此时队列尚未初始化，那就初始化一下咯
        if (t == null) {
            // 注意，同样存在并发，所以需要通过CAS
            if (compareAndSetHead(new Node()))
                tail = head;
            // 可以看到，初始化本身是延迟到有节点入队的时候才进行的
            // 初始化也很简单，就是创建一个空节点作为头部，同时，初始化尾部
            // 注意，初始化之后没有return，所以重新循环会进入到else中
        } else {
            // 这一步就跟addWaiter里面的操作一样啦
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

到目前为止，完成了三个操作

1. 尝试直接获取锁，如果可以，那就获取到，接着按照线程的处理逻辑走
2. 如果获取失败，那将自己挂到等待队列尾部
    1. 如果队列未初始化，那就先初始化一下
    2. 如果挂到队列尾部失败，那就一直挂，直到成功，失败的原因只有一个，被其他人抢了，那最后一定会入队成功的，当没有其他人抢占入队的时候，自己就能入队了

经过前面的几个步骤，此时线程已经入队成功了，此时线程面临着两个选择

- 如果入队完了之后，刚刚好排到自己去运行了，那就不用挂起了，直接去执行就完事了
- 另一种就是，还没轮到自己去运行，那就老老实实挂起了

> acquireQueued

```java
// 返回true表示挂起过程发生中断，false表示没有
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取前一个节点
            final Node p = node.predecessor();
            // 如果前一个节点是head，并且能拿到锁
            if (p == head && tryAcquire(arg)) {
                // 将自己设置为头结点，直接返回
                setHead(node);
                p.next = null;
                failed = false;
                return interrupted;
            }
            // 到了这里，说明要么不是头结点，要么就抢不过别人
            // 那就看下是否要挂起了，要就挂起了
            //  当shouldParkAfterFailedAcquire返回false时，就接着循环
            //   返回true，就进入park了
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // park返回true表明需要中断，记录下中断信号
                interrupted = true;
        }
    } finally {
        // 注意这里，正常走流程是不会到这里的
        // 如果走到这里，说明是抛出异常了，那就取消了
        // 毕竟执行过程出错了
        if (failed)
            cancelAcquire(node);
    }
}
```

> shouldParkAfterFailedAcquire

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 如果前一个节点的ws为SIGNAL，说明当前节点已经向前节点
    // 告知自己需要被唤醒了，那接着挂起吧
    if (ws == Node.SIGNAL)
        return true;
    
    // 如果前一个节点的ws>0，说明前一个节点取消等待了
    // 那只能重新找一个新的前向节点了
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 到了这里说明此时ws=0或者-3
        // 那重新修正一下，修改为SIGNAL(-1)
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    // 刚刚修改完状态或者调整父节点，本次先不挂起
    // 下一波再试试看能不能直接运行，实在不行再挂起，毕竟挂起的代价还是蛮高的
    return false;
}
```

> parkAndCheckInterrupt

```java
private final boolean parkAndCheckInterrupt() {
    // 到这里就挂起了
    LockSupport.park(this);
    // 到这里线程就醒来
    //  醒来的原因有两个
    //   1. 收到中断信号
    //   2. 其他线程执行unpark操作(也就是signal啦)
    return Thread.interrupted();
}
```

> cancelAcquire

```java
private void cancelAcquire(Node node) {
    if (node == null)
        return;

    node.thread = null;

    // 如果前面也有节点取消了，那顺便清理一下吧
    // 毕竟取消了就没有排队的意义了
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    // 标志为取消状态
    node.waitStatus = Node.CANCELLED;

    // 到了这里，当前节点前面的取消了排队的节点就已经被清理了
    
    // 如果node为尾节点，那将pred设置为新的tail
    // 由于此时对tail操作，所以存在竞态(可能此时有人同时入队)
    if (node == tail && compareAndSetTail(node, pred)) {
        // 将pred的next设置为null，也就是tail的next清空啦
        // 原来是引用node嘛
        compareAndSetNext(pred, predNext, null);
    } else {
 		// 如果进入这里，说明取消排队的节点位于队列中间，也就是前后都有人
        // 如果位于中间，那就得看下需不需要通知后面的节点了
        
        int ws;
        // if的条件很严苛，只有同时满足三个，才会进入
        //  进入的目的就是把node后面的节点挂在起来，毕竟node本身已经取消了
        if (
            // 1. node的前节点不是head
            pred != head &&
            // 2. 前节点的状态值=-1或者<=0并且修改为-1成功
            //   这一点是避免执行到if的时候，pred自己也取消了(取消的waitStatus是-1)
            //   
             ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            // 3. pred存在线程，这个应该只是额外检查，大部分情况下都是存在的
            pred.thread != null) {
            Node next = node.next;
            // 将pred的next设置为next，也就是跳过了node了
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            // 不满足上面情况的，那就直接唤醒后面的节点了
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

> unparkSuccessor

```java
private void unparkSuccessor(Node node) {
    
    // 将node状态改为0，表明即将唤醒后面的节点
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
 	// 如果next节点为空或者已经取消排队
    // 那就从后往前找了(这种情况还是很少的，大部分情况都是正常的节点)
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 为什么需要从后往前呢
        //  当s为null时，很显然了，next链已经断开，只能从prev入手了
        // 如果已经取消排队了呢？
        // 回顾一下cancelAcquire的最后一行node.next = node
        // 此时next链(可能)也已经断开了
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

到了这里，获取锁的完整过程就接近分析完毕了，重新回顾一下acquire方法

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 如果发送中断了，那就发送一个中断信号，毕竟之前的已经取消了
        selfInterrupt();
}
```

##### 可中断

前面我们详细分析了acquire方法，该方法是不可中断的，从代码中也可以看到，当发生中断时只是记录下中断信号，然后重新等待获取锁

而可中断的获取锁的方式就不同了，当发生中断的时候，会直接抛出中断异常，从而结束当前的等待，具体看下面分析

> acquireInterruptibly

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    // 先检查一下是否需要中断
    if (Thread.interrupted())
        throw new InterruptedException();
    // 同样的，先试一下，如果失败，再接着走
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

> doAcquireInterruptibly

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    // 包装成Node
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    // 下面的处理逻辑基本同acquireQueued，只是现在响应中断了
    try {
        for (;;) {
            final Node p = node.predecessor();
            // 跟前面一样，拿到锁就结束
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            // 检查是否需要挂起，注意这里发生中断之后，直接就抛出
            // 中断异常了
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

有了前面的铺垫，分析可中断的情况就简单得多了，从代码中也可以看出，不可中断和可中断的区别就在于，可中断在发生中断之后，就取消排队(注意是在finally里面，一定会执行的)，然后抛出异常了

到这里，获取锁的全部内容就分析完毕了，如果没怎么看懂的话，再结合源码多看几次就大致能看懂了

#### 获取共享锁

共享锁的相对于独占锁来说，支持同时多个线程持有锁，这也就使得其稍微有点复杂，接下来详细分析共享锁，同样的，共享锁支持不可中断以及可中断两种形式

##### 不可中断

> acquireShared

```java
public final void acquireShared(int arg) {
    // 同样的，先试下，不成功的话，就接着往下走
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

> tryAcquireShared方法返回类型是一个int，有三种含义
>
> - 负数，表示获取失败
> - 0，以独占形式获取
> - 正数，获取成功

> doAcquireShared

```java
private void doAcquireShared(int arg) {
    // 包装成worker，注意模式是SHARE
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 如果是头节点
            if (p == head) {
                // 先尝试一下，说不定能成功呢
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 到这里说明获取成功了
                    // 设置头部并且尝试唤醒下一个节点
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    
                    // 如果有中断，那传递一下中断信号
                    if (interrupted)
                        selfInterrupt();
                    
                    failed = false;
                    return;
                }
            }
            // 这一部分就跟获取独占锁处理逻辑一样了
            // 检查是否需要挂起，需要就挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
            // 这里需要特别注意
            // 唤醒之后，被唤醒的节点的前一个节点一般就是head，所以，会接着唤醒后续节点
        }
    } finally {
        // 同样的，失败就取消排队
        if (failed)
            cancelAcquire(node);
    }
}
```

从上面的方法可以看到，共享锁的获取方式跟独占锁还是挺接近的，区别在于共享模式中，节点获取到锁之后，会传递信号，具体分析如下

> setHeadAndPropagate

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    // 注意这个操作，将自己设置为头节点
    setHead(node);
    // 如果propagate > 0或者原有的head的状态是<0
    // 或者新的head的状态是<0
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 如果s是共享模式
        if (s == null || s.isShared())
            // 唤醒下一个节点，此时，下一个节点的prev就是当前节点
            // 而前面已经将当前节点设置为head，所以，很大概率下个节点
            // 也会被唤醒(如果能拿到锁)
            doReleaseShared();
    }
}
```

> doReleaseShared

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果head的状态为-1，则改为0，并且唤醒下一个节点
            if (ws == Node.SIGNAL) {
                // 这里失败的原因在于可能多个人同时修改head的状态
                // 比如当前线程以及唤醒当前线程的线程
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                unparkSuccessor(h);
            }
            // 如果head状态为0，则将其改为PROPAGATE
            // 这里就不是很理解，改成PROPAGATE的意义是什么???
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        // 如果头节点没有发生变化，说明唤醒之后，后续节点
        // 还没有调度执行，说明此时任务还是比较繁重，那就退出
        // 当前线程唤醒其他线程的任务了
        // 这里还是不太明白？？？
        if (h == head)
            break;
    }
}
```

从这里可以看到，在获取到锁之后，每个线程都会尝试唤醒下一个节点，而每个线程被唤醒之后，都会检查自己前面是否是头结点，能否获取到锁，如果都可以，就会继续唤醒接下来的线程

这是与独占模式最大的区别，共享模式下，一旦释放锁，会接着尝试唤醒等待队列中处于共享模式的其他节点，注意，是每个被唤醒节点都会这么做，这就相当于一个唤醒链了

##### 可中断

看完了独占锁的可中断模式，很容易就能猜到共享锁的可中断模式了

> acquireSharedInterruptibly

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    // 先检查中断情况
    if (Thread.interrupted())
        throw new InterruptedException();
    // 拿不到锁，那就排队挂起
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

> doAcquireSharedInterruptibly

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
            	// 其实就这里变化了，变成了直接抛出中断异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 释放锁

前面的获取锁部分，详细分析了获取锁的过程，有获取锁，自然就有释放锁了，锁的释放过程相对于获取锁来说，就简单地多了，毕竟释放锁本身并不需要进行同步，接下来详细分析释放锁的过程

同样的，独占锁和共享锁的释放过程是不同了，所以下面也是区分开两者进行分析

#### 释放独占锁

独占锁的释放本身来说，比共享锁简单，因为能够释放独占锁，本身就意味着自己已经持有独占锁了，那一切的操作都是无需关注并发问题的

独占锁的释放主要包含两个流程

1. 释放自己占据的资源(即锁的状态)
2. 唤醒后续节点

> release

```java
public final boolean release(int arg) {
    // 尝试释放，释放成功就唤醒后续节点
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

同样的，`tryRelease`本身是一个空方法，由子类实现，关于如何使用，会在后续分析基于AQS的工具中具体分析到，这里先挖个坑

#### 释放共享锁

释放共享锁相对于独占锁来说，稍微有点复杂，但如果仔细看前面的获取共享锁部分，那其实释放锁的过程基本也就清楚了

> releaseShared

```java
public final boolean releaseShared(int arg) {
    // 尝试释放一下
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

> doReleaseShared

这个方法是实际释放锁的操作，不过这里就不分析了，为啥？前面分析获取共享锁已经分析过了，如果忘记了，返回去看一下就可以了

### ConditionObject

如果你看到这里，首先应该恭喜你，关于AQS的大部分内容基本已经分析完毕了，AQS中最最复杂的内容已经结束了，接下来分析的内容可以理解为AQS的扩展部分

在AQS中，提供了条件等待的语义实现，关于条件等待的基础概念，在前面核心概念部分稍微提到了，接下来将详细分析其具体语义及AQS的实现

#### 条件等待

条件等待其实就是等待指定条件的发生，当线程在处理某个任务的之后，需要特定的资源/条件下才能接着进行，那么，线程就应该停止下来等待该条件的发生，当然，对于开发者而言，就需要对应的工具来实现该语义了

在JDK中，提供了最原始的支持，每一个Object对象本身就是一个条件队列，通过Object对象的`wait`方法就可以实现在该条件上进行等待，通过`notify`或者`notifyAll`就可以唤醒在该条件上等待的对象，同时，需要注意，操作这些方法时候，需要获取该条件的监视器锁

AQS之所以也提供条件等待的支持，就是原始的条件等待本身不够强大，具体表现为：一个Object对象有唯一一把监视器锁，唯一一个等待队列，但在实际使用过程中，却经常需要有同一把锁对应多个条件的情况，如，在操作缓冲区的时候，需要获取缓冲区的锁，然后进行操作(读/写)，此时，缓冲区满了以及缓冲区有空间，对应的是两个条件，但是对应的是操作缓冲区这把锁，通过原始的实现方法，就得通过两个锁来实现两个条件等待了，当条件越多的时候，需要的锁数量也就越多，锁越多，就越不好维护，万一一不小心忘记释放了呢

所以，在JDK1.5中，提出了`Condition`的概念，同时，通过AQS提供实现

`Condition`的语义与原始的条件等待是一样的，但解决了原始条件等待存在的问题，对于`Condition`的操作，同样需要获取对应的锁，只是`Condition`不再依赖于监视器锁，而是通过自定义锁提供支持，如`ReentrantLock`，同一把锁可以支持一个到多个`Condition`，从而满足了在同一把锁上实现多个条件等待的需求

> Condition

Condition接口提供的方法都非常清晰，简单看一下即可

```java
public interface java.util.concurrent.locks.Condition {
  public abstract void await() throws InterruptedException;
  public abstract void awaitUninterruptibly();
  public abstract long awaitNanos(long) throws InterruptedException;
  public abstract boolean await(long, TimeUnit) throws InterruptedException;
  public abstract boolean awaitUntil(Date) throws InterruptedException;
  // 对应Object的nofity和notifyAll
  public abstract void signal();
  public abstract void signalAll();
}
```

`Condition`中的`await`对应的就是`Object`中的`wait`，`signal`对应`notify`，`signalAll`对应`notifyAll`，至于为什么不直接覆盖，应该是为了保留原始的语义，`Condition`本身也可以作为一个条件(一切均继承`Object`)

了解了AQS加锁的过程，大致也可以猜出来，等待条件发生的过程其实就是将等待的节点挂到指定的条件队列上，而条件的到达，其实就是对应的操作了

接下来详细分析AQS中`Condition`语义的实现，即`ConditionObject`

#### ConditionObject

##### ConditionObject结构

ConditionObject也是AQS的内部类，包含的属性也是比较简单的

- `firstWaiter`
- `lastWaiter`

两个属性的类型都是`Node`，就是上面已经分析过的那个`Node`，看到这里，大致上就可以猜出来了，AQS中的`Conditoin`实现其实就是一个链表，将等待的节点挂到该链表上而已

##### 等待

等待的语义就是说，当前这个条件不满足接着运行下去的需求，那么需要挂起等待其条件的发生，当然，由于自己已经挂起了，那么唤醒自然就是由其他人来操作了

当然，等待本身也是支持不可中断已经可中断的，从前面分析获取锁的过程大致已经可以得知，两者差异不大，这里我们就放一起分析了，不单独开一个小节

> await

```java
public final void await() throws InterruptedException {
    
    if (Thread.interrupted())
        throw new InterruptedException();
    
    // 将自己添加到条件队列中(先看下面对该方法的分析，再接着往下走)
    Node node = addConditionWaiter();
    // 释放自己占据的锁，必须先获取锁才能进行等待(先看下面对该方法的分析，再接着往下走)
    int savedState = fullyRelease(node);
    
    // 到这里就说明锁释放成功了，那接下来就准备挂起，等待条件发生了
    
    int interruptMode = 0;
    
    // 检查看自己是否在同步队列(等待队列)
    //  如果不在，那就说明自己还没有被唤醒，此时醒来是意外
    //  为什么呢？
    // 联想一下前面获取锁的操作，要执行条件等待，必须先获取锁，此时自己必定是队头
    // 同时，等待之前，必须释放锁，此时会唤醒等待队列的后续节点，那么，也就意味着自己已经
    // 离开了等待队列，在条件队列中了
    // 如果又回到了等待队列，那么就说明情况发生了，自己被重新移动回等待队列中(参加唤醒操作)
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        // 检查是否在等待过程中出现中断(先看下面对该方法的分析，再接着往下走)
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 直接滑动到checkInterruptWhileWaiting的分析之后，这里太庞大了，放在后头分析
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

> addConditionWaiter

```java
// 将等待节点添加到条件队列中，该方法可以说是条件等待的核心了
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果有节点取消等待了，那就先移除咯
    // 取消了就没必要等待了
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 比较直观，将自己挂到队尾就行了
    // 这里之所以不需要通过CAS，是因为此时已经持有锁了，不存在并发情况
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

> unlinkCancelledWaiters

```java
// 将取消排队的节点移除
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    // 从头到尾扫一遍，如果取消了，就将其移除
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            // 断开t
            t.nextWaiter = null;
            // trail为null说明中间没有出现跳点，即(取消-不取消-取消的情况)
            if (trail == null)
                firstWaiter = next;
            // 跳过取消的节点
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        // 如果节点依旧在等待，trail往后移动就行
        else
            trail = t;
        t = next;
    }
}
```

> fullyRelease

```java
// 完全释放锁，毕竟有重入的情况存在
// 返回当前锁的情况，后续重新获取的时候需要使用到
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取当前锁状态(包含重入的情况)
        int savedState = getState();
        // 如果能够释放成功，那就返回了
        // release的行为取决了tryRelease
        // 正常情况来说，释放都是成功的，毕竟是自己持有锁
        if (release(savedState)) {
            failed = false;
            return savedState;
        // 如果释放失败，那就抛出监视器异常
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        // 如果当前节点失败，将当前节点状态调整为取消，后面会被移除
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

> isOnSyncQueue

```java
final boolean isOnSyncQueue(Node node) {
    // 如果此时状态还是等待(见signal分析)，或者前面没有节点
    // 说明还没有在等待队列中(思考一下进入等待队列的操作，先修改prev，再修改next)
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 如果next已经有值了，那就说明一定在等待队列中了
    // (思考一下进入等待队列的操作,先修改prev，再修改next)
    if (node.next != null)
        return true;
    // 从后往前找一下，看是否存在于等待队列中
    // 同样需要结合入等待队列的操作分析
    return findNodeFromTail(node);
}
```

> findNodeFromTail

```java
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

> checkInterruptWhileWaiting

```java
// 检查是否在等待条件过程中发送中断
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        // 如果发生中断，看下是在哪个环节发送的中断
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

> transferAfterCancelledWait

```java
final boolean transferAfterCancelledWait(Node node) {
    // 此时已经唤醒了，修改一下状态
    // 此时有两种情况，一种是通过signal唤醒，那么状态会被signal修改,CAS失败
    // 如果CAS成功，意味着是非正常唤醒，有可能是中断，有可能是系统偶然唤醒
    // 但此时已有中断信号，说明只能是被中断了
    // 注意，此时CAS是成功的
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        // 如果被意外唤醒，那进入到等待队列排队吧
        enq(node);
        return true;
    }
    // 到这里说明是正常唤醒，如果还没有在等待队列中，说明signal操作还没有完成
    // 自旋一下等待完成就可以了
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

接着回到最开始的`await`操作

> await

```java
 public final void await() throws InterruptedException {
   // .....
     while (!isOnSyncQueue(node)) {
         // 到这里就挂起了
         LockSupport.park(this);
         // 到这里就被唤醒了
         //  1.通过signal唤醒
         //  2.中断唤醒
         
         // 前面分析到这里，checkInterruptWhileWaiting会返回三种值
         //  0，没有发生中断，那么就继续检查，排队吧
         //  如果不是0，说明发生了中断，那就该起来工作了
         if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
             break;
     }
     
     // 注意，此时只是被唤醒，还没有拿到锁
     
     // 重新排队等待获取锁
     if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
         interruptMode = REINTERRUPT;
     // 到这里说明获取锁，顺便清理一下取消的节点
     if (node.nextWaiter != null)
         unlinkCancelledWaiters();
     // 如果发生了中断，那就处理一下
     if (interruptMode != 0)
         reportInterruptAfterWait(interruptMode);
 }
```

> reportInterruptAfterWait

```java
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    
     //  THROW_IE(-1)，transferAfterCancelledWait返回true则返回该值
     //    说明此时是被中断唤醒，那就需要抛出中断异常了
     //  REINTERRUPT(1)，通过signal正常唤醒
    
    // await是可中断的
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    
    // 发送下中断信号就行
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

到这里，等待条件的操作就结束了，重新梳理一下await操作的流程

1. 将自己添加到条件队列中
2. 释放占据的锁
    1. 释放失败说明是非法操作
3. 等待唤醒
    1. signal唤醒
    2. 中断唤醒
4. 唤醒之后，等待获取锁，获取之后
    1. 接着挂起之前的流程
    2. 或者需要抛出异常，则直接抛出中断异常

##### 唤醒

前面详细分析了等待条件发生的具体操作，接下来分析唤醒的具体操作

> signal

```java
public final void signal() {
    // 如果不是排它锁的持有者，则直接抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    // 唤醒下一个节点
    if (first != null)
        doSignal(first);
}
```

> doSignal

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // first即将唤醒，断开连接
        first.nextWaiter = null;
        // 将找到的第一个合适的节点从条件队列移动到等待队列
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

> transferForSignal

```java
// 将节点移动到等待队列，等待唤醒
final boolean transferForSignal(Node node) {

    // 如果修改状态失败，说明中途已经被唤醒了
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 入队
    Node p = enq(node);
    int ws = p.waitStatus;
    // 如果前状态已经取消排队，修改前节点表示自己需要被唤醒
    // 如果修改失败，那就说明前节点状态已经发送了变化，把自己也唤醒
    // 由于已经入队了，所以会去参与竞争锁
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

可以看到，唤醒的操作也是比较直观的，将第一个正常的节点移动到等待队列中等待唤醒，或者直接唤醒该节点(特殊情况)

> signalAll

`signal`操作只是唤醒其中的一个，如果需要唤醒多个，则通过`signalAll`

```java
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}
```

> doSignalAll

```java
// 将所有节点都尝试移动到等待队列，等待获取锁
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

到这里，唤醒的操作分析就结束了，稍微总结一下唤醒的过程

1. 将节点移动到等待队列
2. 根据情况决定唤醒该节点还是结束本次唤醒操作(等待等待队列头节点唤醒即可)

## 总结

到了这里，基本上AQS的核心功能就分析完成了，当然，还有少部分的内容没有分析，这并非偷懒或者遗忘，而是这一部分的内容与后面的内容更加贴切，所以就将其放在后面分析了

在本文中，详细对AQS进行了详细的介绍，AQS是一个并发安全的同步器，通过AQS，可以简单快捷地实现一个并发安全的工具，当然，本文中只是提出这点，但并没有具体体现，在后面一篇文章中将结合已有基于AQS实现的同步在体现AQS的这个特点

在本文中，我们详细分析了AQS的获取锁的方式，包括

- 独占锁
    - 不可中断
    - 可中断
- 共享锁
    - 不可中断
    - 可中断

同时详细分析了其释放锁的具体操作以及条件队列的实现`ConditionObject`

从分析中可以看出，AQS是基于链表来实现的，等待队列是基于双向链表，而条件队列是基于单向链表。如果获取不到锁或者条件不满足，则将其放到对应的队列尾部，等待唤醒，条件队列的节点唤醒之后，会将自己从条件队列移动到等待队列，等待获取锁

通过对AQS的分析，可以更好地理解一些并发工具的特性，在使用这些工具的时候能让我们更加清楚这些工具的注意事项

好了，到这里，对AQS的分析就告一段落了，后面将详细分析基于AQS实现的几个同步工具，Let's keep going on.

