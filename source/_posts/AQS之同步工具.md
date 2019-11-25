---
title: AQS之同步工具
date: 2019-11-25 14:28:00
tags:
  - Java
  - AQS
  - ReentrantLock
  - CountdownLatch
  - Semaphore
  - CyclicBarrier
categories:
  - Java
  - Concurrent
---

在前面一篇文章中，详细分析了AQS的原理，但没有涉及到AQS的使用，对于AQS我们也只是知其所以然而不止其然，只知道AQS长什么样，有什么用，是怎么实现的，但却缺少了最核心的，AQS要怎么用

在这篇文章中，我们将详细分析众多基于AQS实现的同步工具，如`ReentrantLock`、`CountdownLatch`等，通过这些工具的实现来了解AQS的使用，从而打通AQS使用、分析的整个过程

<!--more-->

# AQS之同步工具

## 前言

前面提到了，`java.util.concurrent`包中的很多同步工具都是基于AQS来实现，常用的工具有

- ReentrantLock
- CountdownLatch
- Semaphore
- CyclicBarrier
- ...

这篇文章将对上面列出的这四个工具进行详细分析

这四个工具大体上可以分为两类，其中ReentrantLock代表的是独占锁模式，另外三个或多或少代表的则是共享锁模式，分别对应了AQS的两种模式

对于这四个工具的分析，会分成两部分，第一个部分是该工具的介绍、使用、应用场景，第二个部分则是这篇文章的重点，即该工具的源码分析，希望这篇文章能对你有所帮助

## ReentrantLock

### Lock

`java.util.concurrent.locks.Lock`接口是JDk1.5中引入的，提供与`synchronized`相同的锁的语义，即限制资源的访问，但两者有所区别(废话，没区别就不需要增加了)

两种锁各有千秋

`synchronized`的优点在于使用简单、方便，本身由JVM控制锁的获取及释放，所以开发者不需要担心忘记释放锁的情况，但是其不足也是比较明显

- 仅支持阻塞模式，即锁要么获取成功，要么阻塞直到获取成功
- 不支持超时
- 不支持中断

`synchronized`的不足就是`Lock`接口所需要补充的了，Lock接口提供的功能如下

```java
public interface Lock {
  // 不支持中断的获取锁，与synchronized使用一致
  public abstract void lock();
  // 支持中断的获取锁
  public abstract void lockInterruptibly() throws InterruptedException;
  // 仅尝试获取锁，如果成功，则返回true，如果获取失败，则返回false，不阻塞
  public abstract boolean tryLock();
  // 同上，但是在指定时间内如果能获取到，则成功返回true，失败返回false，中断则抛出异常
  public abstract boolean tryLock(long, TimeUnit) throwsInterruptedException;
  public abstract void unlock();
  public abstract Condition newCondition();
}
```

从上面的方法结合前面的分析，大概就能猜出来Lock所提供的功能了

### ReentrantLock

`ReentrantLock`是`java.util.concurrent.locks.Lock`接口的实现

本身提供的重入的语义，并且支持公平模式以及非公平模式

所谓重入，一把锁可以多次被同一个线程持有，即该锁可以被该线程“重入”

重入基本上是必须的，比如有两个被同一个锁限制访问的方法，如果锁本身不具备重入的话，那该线程将无法在其中一个方法访问另一个方法，因为在访问第一个方法的时候，已经获取了锁，而此时还没有释放锁就去访问第二个方法，由于锁本身不重入，所以要求该线程重新获取锁，但该锁已经被获取了，所以无法再被获取

注意，`synchronized`提供的也是具有重入语义的锁

#### 使用

既然是锁，那使用其实也就非常直观了，在进入临界区之前，获取锁，退出临界区之后，释放锁

接下来看下其具体使用

```java
class Task implements Runnable {

    private Lock lock;

    public Task(Lock lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        Thread thread = Thread.currentThread();
        log.info("thread:[{}] try to get lock", thread);
        try {
            // 获取锁
            lock.lock();
            log.info("thread:[{}] get the lock", thread);
            Random random = new Random();
            try {
                TimeUnit.SECONDS.sleep(random.nextInt(10));
            } catch (InterruptedException ignored) {}
        }finally {
            log.info("thread:[{}] release the lock", thread);
            // 释放锁
            lock.unlock();
        }
    }
}
```

如果同时传入一把锁，在多线程环境下执行该任务，可以看到可能同时多个线程打印出`try to get lock`，但只会有一个线程打印出`get the lock`，打印出`release the lock`，即释放锁只有，其他线程才有可能获取到锁

重入锁的使用很简单，但需要注意，一定要保证获取到的锁在不需要使用的使用能够释放，`Doug Lea`在Lock接口的doc中也推荐在`try-finally`块中使用，并且在`finally`中释放，这样可以保证无论是否执行过程中抛出异常，一定能够释放该锁(finally)一定会执行

其他尝试获取锁的方式也是类似的，这里就不举例了

#### 分析

接下来就是这一小节的重头戏了，我们将挨个分析ReentrantLock的方法

##### 签名及属性

ReentrantLock的签名

```java
public class ReentrantLock implements Lock, java.io.Serializable {
}
```

属性

```java
private final Sync sync;
```

Sync是ReentrantLock的内部类，继承了AQS，在AQS的doc中明确说明，所有AQS的子类应该尽量封闭在工具类的内部，避免被外部直接访问

##### 方法

> 构造方法

```java
// 默认情况下是非公平模式
public ReentrantLock() {
    sync = new NonfairSync();
}
```

```java
// 传入true则是公平模式，false是非公平模式
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

> 获取锁

```java
// 不可中断
public void lock() {
    sync.lock();
}
```

```java
// 可中断
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

```java
// 尝试获取锁，注意这个哦，不管是公平锁还是非公平锁
// tryLock都是直接非公平的，毕竟就试一下
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
```

```java
// 带超时时间的尝试获取锁
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

> 释放锁

```java
public void unlock() {
    sync.release(1);
}
```

> 创建condition

```java
public Condition newCondition() {
    return sync.newCondition();
}
```

> 查看当前线程持有锁数量

```java
public int getHoldCount() {
    return sync.getHoldCount();
}
```

> 查看当前线程是否是排它锁持有者

```java
public boolean isHeldByCurrentThread() {
    return sync.isHeldExclusively();
}
```

> 是否锁住

```java
public boolean isLocked() {
    return sync.isLocked();
}
```

> 是否是公平锁

```java
public final boolean isFair() {
    return sync instanceof FairSync;
}
```

> 获取属主

```java
protected Thread getOwner() {
    return sync.getOwner();
}
```

> 查看是否还有线程在排队

```java
public final boolean hasQueuedThreads() {
    return sync.hasQueuedThreads();
}
```

> 查看等待队列长度

```java
public final int getQueueLength() {
    return sync.getQueueLength();
}
```

> 查看还在等待的线程

```java
protected Collection<Thread> getQueuedThreads() {
    return sync.getQueuedThreads();
}
```

> 查看某一条件下是否还有线程在等待

```java
public boolean hasWaiters(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
}
```

> 查看条件队列长度

```java
public int getWaitQueueLength(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
}
```

> 获取在条件队列上等待的线程

```java
protected Collection<Thread> getWaitingThreads(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
}
```

从上面的方法在可以看到，几乎所有的操作都是委托给了sync对象，这也就意味着真正的操作其实是sync

在接着往下看之前，我们先来回忆一下AQS，AQS中提供了众多的`tryXXX`方法，用于子类实现对应的尝试获取

操作，也就是说，子类决定了能否获取锁，如果不能，那么剩下的就是AQS的事情了，同样的，释放锁的操作也是如此

##### Sync

Sync是ReentrantLock的内部类，继承了AQS，提供了ReentrantLock的所有功能，有两个子类，`FairSync`和`NonfairSync`，分别提供公平锁和非公平锁的实现

Sync是一个抽象类，把公平锁跟非公平锁公有的逻辑都实现了，大部分的功能都是直接调用AQS的方法来实现

Sync本身的代码不长，我们就直接分析了

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    abstract void lock();

    // 非公平锁的获取
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 如果没有人获取
        if (c == 0) {
            // 既然是非公平，那就不管有没有人排队了，直接上
            // 能抢到就是我的了
            if (compareAndSetState(0, acquires)) {
                // 设置自己为排它锁所有者
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果是重入，那就直接获取到了
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) 
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 获取失败
        return false;
    }

    // 尝试释放锁
    // 公平锁跟非公平锁的释放是一样的，所以在父类实现了
    // relase会先调用tryRelase(),try成功了就唤醒下一个节点
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        // 如果当前线程不是排它锁的获取，那就是非法操作了
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 如果c==0，意味着锁可以完全释放了，否则就是重入
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        // 重新设置下锁状态
        setState(c);
        return free;
    }

    // 当前线程是否是排它锁的获取者
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    // 获取Condition对象
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // 获取当前锁的属主
    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }
    // 获取当前持有的资源数，只有是排它锁的持有者才能获取得到
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    // 锁是否已经被获取了
    final boolean isLocked() {
        return getState() != 0;
    }

    // 用于序列化的
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

有了前面AQS的分析，这里分析起来就轻松多了

##### FairSync

```java
// 公平锁实现
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    // 可以看到，lock方法其实就是调用了AQS的acquire方法
    // 回忆一下，AQS的acquire会先调用tryAcquire
    final void lock() {
        acquire(1);
    }

    // tryAcquire
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 没有人在执行
        if (c == 0) {
            // 既然是公平，那就得看一下有没有人在排队了
            // 如果有，那获取失败，老老实实排队去
            // hasXXX是来自AQS的方法，见下面分析
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

> hasQueuedPredecessors

```java
// 公平锁实现的核心，就是先看下有没有人在排队
public final boolean hasQueuedPredecessors() {
    Node t = tail; 
    Node h = head;
    Node s;
    // 如果h != t说明队列有元素，有两种情况
    //   1. head刚刚初始化(回忆一下AQS的enq方法)，有人在执行初始化操作
    //   2. 已经有元素在排队了
    return h != t &&
        // h.next == null说明有人在执行初始化的入队操作，说明有人了
        // s.thread 不是自己，也说明有人了
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

可以看到，有了AQS之后，一个公平锁的实现是非常简单的，只需要实现一个`tryAcquire`一个`tryRelease`就可以了

##### NonfairSync

前面我们提到，ReentrentLock默认的实现就是非公平锁，在看Sync的时候，我们也看到了一个名为`nonfairTryAcquire`的方法，隐约中已经可以猜测得到该方法就是非公平锁获取锁的实现了

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    // 非公平，那就先抢一下，抢到了就是我的了
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        // 抢不到就调用acquire
        else
            acquire(1);
    }

    // 直接通过nonfairTryAcquire
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

到这里，非公平锁的分析就结束了，有木有一种挖槽的感觉，这实现居然如何简洁

#### 小结

ReentrantLock是JDK1.5提供的另一种锁的实现，支持重入，支持公平锁、非公平锁。

其实际操作是委托给内部类 Sync，Sync是基于AQS实现的同步对象，包含两个子类，`FairSync`、`NonfairSync`，分别对应的就是公平锁、非公平锁

从这里我们就可以看出了，对于一个锁的实现，只需要继承AQS，然后实现其`tryLock`、`tryRelease`方法就足够了，这也是AQS设计的初衷

## CountdownLatch

`CountdownLatch`同样是JDK1.5提供的并发工具，不过用途则与`ReentrantLock`不同

`CountdownLatch`翻译过来是栅栏，用于是实现一部分线程等待另一部分线程完成工作，然后再继续往下执行任务的场景，最典型的用途就是异步转同步

### 使用

下面的例子演示了如何使用CountdownLatch来实现线程相互等待(异步转同步)的功能

```java
@Slf4j
public class CountdownLatchTest {

    public static void main(String[] args) {
        int threadNum = 5;
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch finishLatch = new CountDownLatch(threadNum);

        for (int i = 0; i < threadNum; i++) {
            Thread thread = new Thread(new Task(startLatch, finishLatch));
            thread.start();
        }
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException ignored) {}

        log.info("main: everyone start working....");
        // 开始打开栅栏
        startLatch.countDown();
        
        log.info("main: waiting for job finish");
        
        try {
            // 等待所有任务完成
            finishLatch.await();
        } catch (InterruptedException ignored) {
        }
        // main的其他工作本来与其他任务是并行，现在只有工作任务完成了才会进行到这里
        // 也就是将原本异步的任务转为了同步
        
        log.info("main: all job done");
    }

    static class Task implements Runnable {

        private CountDownLatch startLatch;
        private CountDownLatch finishLatch;

        public Task(CountDownLatch startLatch, CountDownLatch finishLatch) {
            this.startLatch = startLatch;
            this.finishLatch = finishLatch;
        }

        @Override
        public void run() {
            Thread thread = Thread.currentThread();
            log.info("thread: [{}] is waiting start signal", thread);
            try {
                // 所有线程等待开始
                startLatch.await();
            } catch (InterruptedException ignored) {
            }

            log.info("thread: [{}] start working...", thread);

            Random random = new Random();
            try {
                TimeUnit.SECONDS.sleep(random.nextInt(10));
            } catch (InterruptedException ignored) {
            }

            log.info("thread: [{}] finish working...", thread);
				
            // 线程完成任务
            finishLatch.countDown();
        }
    }
}
```

再举一个例子，调用一个方法，该方法通过线程去拉取数据，只有拉取数据任务结束，该方法才返回，这个时候`CountdownLatch`就非常重要了，只需要启动线程之后，执行`await()`等待，线程完成之后通过`countdown()`打开栅栏就可以实现我们的想法了，简洁而高效

### 分析

按照老惯例，分析下CountdownLatch的实现，通过上面的说明，我们已经知道了是基于AQS实现的，并且，可以被多个线程同时持有，那就是共享模式无疑了

#### 签名及属性

签名

```java
public class CountDownLatch {
}
```

属性

```java
private final Sync sync;
```

看到这里，是不是有种似曾相似的感觉，跟ReentrantLock灰常相似

#### 方法

> 构造方法

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

> 等待

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

> 带超时时间的等待

```java
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

> 打开闭锁

```java
public void countDown() {
    sync.releaseShared(1);
}
```

> 获取当前计数

```java
public long getCount() {
    return sync.getCount();
}
```

CountdownLatch锁提供的方法相对来说比较少，可以看到，基本上也是委托给了sync

在接着往下之前，先回忆一下，AQS中，共享模式跟独占模式是很相似的，都是要求子类实现`tryXX`方法，不同的是，共享模式是实现`tryXXXShared`而独占模式是实现`tryXX`

#### Sync

```java
// 同样的，继承AQS
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    // 使用count来初始化AQS
    Sync(int count) {
        setState(count);
    }

    // 获取数量，其实就是获取state的值
    int getCount() {
        return getState();
    }

    //回忆一下，AQS中acquireShared/acquireSharedInterruptibly会到调用该方法
    // 该方法返回>0表示可以获取，<0表示获取失败
    // 结合前面await，可以猜到，当state==0时，await操作成功，即“获取到锁”
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    // 释放，再回忆一下AQS的releaseShared
    protected boolean tryReleaseShared(int releases) {
        // 自旋释放
        for (;;) {
            int c = getState();
            // 如果c==0，说明是独占形式了，不能释放
            if (c == 0)
                return false;
            int nextc = c-1;
            // 如果资源个数为0，说明可以释放锁了
            // 释放之后，就会执行unpark操作了，从而将await的线程唤醒
            // 由于是共享，所以所有await的线程都会被唤醒
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

到了这里CountdownLatch的分析就结束了，可以看到，有了AQS，一切的操作就变得非常简单了

### 小结

分析完了CountdownLatch之后，我们可以直观地看到，实现一个并发安全的共享类型的同步工具，同样只需要实现`tryAcquireShared`以及`tryReleaseShared`，其他的直接交给AQS就可以了，AQS真好

## Semaphore

Semaphore是我们分析的第三个基于AQS的同步工具

Semaphore，翻译过来是信号量，本身维护了一组凭证，代表可用资源数量，当需要使用的时候，先向Semaphore申请，使用完毕之后归还，如果资源无法满足，则挂起该线程，等待有足够的资源

根据上面的描述，可能会觉得很熟悉，跟锁很相似，确实，凭证数量为1的信号量就退化成了锁了，也可以将Semaphore理解为多元锁

Semaphore是基于共享模式实现的，提供了公平模式与非公平模式的支持

### 使用

Semaphore的使用也是比较简单的，主要就是资源的申请`acquire`以及释放`release`

```java
public class SemaphoreTest {

    public static void main(String[] args) {
        // 凭证总数量为4个
        Semaphore semaphore = new Semaphore(4);
        for (int i = 0; i < 10; i++) {
            new Thread(new Task(semaphore)).start();
        }
    }

    static class Task implements Runnable {

        private Semaphore semaphore;

        public Task(Semaphore semaphore) {
            this.semaphore = semaphore;
        }

        @Override
        public void run() {
            log.info("try to get resource");
            try {
                // 每次申请一个
                // 所以在当前系统中，最多同时有4个任务获取到凭证
                semaphore.acquire();
            } catch (InterruptedException ignored) { }

            log.info("get the resource");

            Random random = new Random();
            try {
                TimeUnit.SECONDS.sleep(random.nextInt(10));
            } catch (InterruptedException ignored) {}

            log.info("finish job, release resource");
            // 归还资源
            semaphore.release();
        }
    }
}
```

### 分析

#### 签名及属性

签名

```java
public class Semaphore implements java.io.Serializable {
}
```

属性

```java
private final Sync sync;
```

咦，好面熟

#### 方法

> 构造方法

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
```

```java
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

permits，即凭证，是必须的，默认情况下是非公平模式，与ReentrentLock相似

> 获取一个凭证

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

> 获取多个凭证

```java
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
```

> 不可中断获取凭证

```java
public void acquireUninterruptibly() {
    sync.acquireShared(1);
}
```

> 不可中断多个凭证

```java
public void acquireUninterruptibly(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireShared(permits);
}
```

> 尝试获取凭证

```java
public boolean tryAcquire() {
    return sync.nonfairTryAcquireShared(1) >= 0;
}
```

> 尝试获取多个凭证

```java
public boolean tryAcquire(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.nonfairTryAcquireShared(permits) >= 0;
}
```

> 带超时时间的尝试获取凭证

```java
public boolean tryAcquire(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

> 超时时间多个凭证

```java
public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
}
```

> 释放凭证

```java
public void release() {
    sync.releaseShared(1);
}
```

> 释放多个凭证

```java
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
```

> 查看可用凭证

```java
public int availablePermits() {
    return sync.getPermits();
}
```

> 缩减凭证

```java
protected void reducePermits(int reduction) {
    if (reduction < 0) throw new IllegalArgumentException();
    sync.reducePermits(reduction);
}
```

> 释放所有凭证，返回当前可用凭证

```java
public int drainPermits() {
    return sync.drainPermits();
}
```

可以看到，其实是满满的套路，所有的操作同样委托给了Sync

#### Sync

Sync是Semaphore中的核心，Sync是一个抽象类，提供了Semaphore中通用的操作

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;

    // 可以看到，通过permits来初始化state
    Sync(int permits) {
        setState(permits);
    }

    final int getPermits() {
        return getState();
    }

    // 非公平锁的获取
    // 一直自旋，直到获取成功或者资源不足
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            // 资源不足，立即返回
            // 资源足够，CAS失败，说明有竞争，继续试一下就行
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    // 尝试释放
    // 自旋，直到释放成功
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

    // 缩减，其实就是重新设置state
    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

    // 清空，其实就是将其设置为0
    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```

Sync就没啥好说的啦，看一下代码大概就清楚了

#### FairSync

```java
// 公平模式
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            // 既然是公平模式，那就先看先有没有人排队
            // 有就说明获取失败，需要排队
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

套路啊套路

#### NonfairSync

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

### 小结

Semaphore提供了一种限制资源获取的方式，只有先获取到对应的凭证，才能接着操作，基于AQS的共享模式实现，提供了公平模式、非公平模式

如果从上一篇文章跟下来，看到这里，大概只有一个想法了，AQS是真的强

## CyclicBarrier

习惯了上面三个工具的分析模式，可能会觉得，CyclicBarrier已经没有分析的必要了，如果这样想，那可能就会错过了很有趣的CyclicBarrier

CyclicBarrier，翻译过来是循环栅栏，对，同样也是栅栏，所以从概念上来说，是与CountdownLatch类似的功能，也是用于线程的同步，只不过，两者的目标不一样

- CountdownLatch主要是针对多个不同任务之间进行协作的，如任务A需要等待任务B完成之后再接着运行
- CyclicBarrier是针对同一类型任务的，只有所有任务都到达某个点之后，这些任务才接着继续执行

需要区分好两者的使用场景

### 使用

下面这里例子，表达的就是前面所举的例子了，所有线程先执行任务，所有到达A点之后，栅栏才会打开，然后执行第二个任务，同时到达之后，完成任务

```java
@Slf4j
public class CyclicBarrierTest {

    public static void main(String[] args) {
        int threadNum = 10;
        // 初始化参与人数，以及每个任务完成之后的操作
        CyclicBarrier barrier = new CyclicBarrier(threadNum, new Runnable() {
            @Override
            public void run() {
                log.info("all job done");
            }
        });
        for (int i = 0; i < threadNum; i++) {
            new Thread(new Task(barrier)).start();
        }

    }

    static class Task implements Runnable {

        private CyclicBarrier cyclicBarrier;

        public Task(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            Thread thread = Thread.currentThread();
            Random random = new Random();
            log.info("thread: [{}] start running...", thread);

            try {
                TimeUnit.SECONDS.sleep(random.nextInt(10));
            } catch (InterruptedException ignored) {}

            log.info("thread:  [{}] at pointA, waiting...", thread);
				
            // 我的第一任务到达，等待其他人完成
            try {
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                log.error("IE", e);
            } catch (BrokenBarrierException e) {
                log.error("broken ", e);
            }

            log.info("thread: [{}] start another job", thread);
            try {
                TimeUnit.SECONDS.sleep(random.nextInt(10));
            } catch (InterruptedException ignored) {}

            // 我的第二任务到达，等待其他人完成
            try {
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                log.error("IE", e);
            } catch (BrokenBarrierException e) {
                log.error("broken ", e);
            }

            // 完成所有任务
            log.info("thread: [{}] finish job", thread);
        }
    }
}
```

从示例中就可以看出，只有一个阶段的CyclicBarrier，其实就是一个CountdownLatch啦

### 分析

#### 签名及属性

签名

```java
public class CyclicBarrier {
}
```

属性

```java
// 惊不惊喜，意不意外，居然不是Sync了
private final ReentrantLock lock = new ReentrantLock();
// 等待其他人完成
private final Condition trip = lock.newCondition();
// 参与人数
private final int parties;
// 所有线程到达之后执行的任务
private final Runnable barrierCommand;
// 当前年代，也称为当前阶段
private Generation generation = new Generation();
```

看到这里，是不是有种被坑的感觉，这明明就没有AQS的身影，怎么可以放在这里呢，咳咳，请注意`ReentrantLock`以及`lock.newConditoin()`

从这里就可以看出，`CyclicBarrier`是通过`ReentrantLock`以及`Condition`来实现的，小样，本质还是一样的，只会比继承AQS更加简单

大致上我们已经可以猜出来了，通过`ReentrantLock`进行访问的限制，通过`Condition`进行条件的等待

> Generation

```java
// 记录阶段的，每个阶段对应一个Generation对象
private static class Generation {
    // 记录当前阶段是否已经被打破了，也即CyclicBarrier是否被打破
    boolean broken = false;
}
```

#### 方法

> 构造方法

```java
// 参与人数以及全体到达之后需要执行的任务
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}
```

> 自己已到达，等待别人完成

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        // false代表不设置超时
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

```java
public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
           BrokenBarrierException,
           TimeoutException {
    // 带超时时间的版本
    return dowait(true, unit.toNanos(timeout));
}
```

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
               
    final ReentrantLock lock = this.lock;
    // 先持有锁
    lock.lock();
    try {
        final Generation g = generation;
		// 判断一下当前CyclicBarrier是否已经被打破了
        // 如果是，则抛出BrokenBarrierException异常
        if (g.broken)
            throw new BrokenBarrierException();

        // 检查一下是否发生了中断
        // 如果发生了中断，则打破栅栏
        // 并且抛出InterruptedException
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        // 到这里的话，说明一切正常
        int index = --count;
        // 如果count等于0，说明最后一个任务已经到达
        // 可以唤醒其他人了
        if (index == 0) {  
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                // 如果有任务需要执行，就由当前线程负责执行
                if (command != null)
                    command.run();
                
                ranAction = true;
                // 为下一轮做准备
                nextGeneration();
                return 0;
            } finally {
                // 如果执行终结任务发送问题
                // 那就打破栅栏
                if (!ranAction)
                    breakBarrier();
            }
        }

        // 到这里，说明要么不是最后一个任务，要么执行终结任务过程出错了
        
        // 一直等待，直到正常结束或者异常退出
        for (;;) {
            try {
                // 还记得这一行代码吗
                //     private final Condition trip = lock.newCondition();
                
                // 如果没有设置超时，那么就一直等待就行了
                if (!timed)
                    trip.await();
                // 如果设置了超时，时间还没到，那就进行超时等待
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 如果等待过程发生中断
                //  1.如果还在当前这一年代，并且没有被打破
                //   那就打破，并且抛出异常
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // 如果已经到了下一代，说明是最后一个任务已经执行完成
                    // 已经有人执行完nextGeneration，睡过了头？？？
                    // 直接发信号告诉当前线程就行了
                    Thread.currentThread().interrupt();
                }
            }
			
            // 如果醒来后发现已经被打破了，那就抛出异常
            if (g.broken)
                throw new BrokenBarrierException();

            // 如果年代已经变了，说明已经正常结束了
            // 那就应该跳出循环了
            if (g != generation)
                return index;

            // 设置了超时时间，并且时间已经过去了
            // 那就是等待超时了
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
     // 还记得这里吗，在finally释放锁哦
    } finally {
        lock.unlock();
    }
}
```

> 调整到下一代

```java
private void nextGeneration() {
    // 唤醒全部等待的线程
    // 此时已经获取了锁，所以不必重新获取了
    trip.signalAll();
    // 重置计数器
    count = parties;
    // 新建一个年代器
    generation = new Generation();
}
```

> 打破

```java
private void breakBarrier() {
    // 打破了
    generation.broken = true;
    count = parties;
    // 唤醒全部
    trip.signalAll();
}
```

> 查看是否已经打破

```java
public boolean isBroken() {
    final ReentrantLock lock = this.lock;
    // 需要先获取锁
    lock.lock();
    try {
        return generation.broken;
    } finally {
        lock.unlock();
    }
}
```

> 重置Barrier

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 先打破
        breakBarrier();  
        // 在进行到下一代，表示重置
        nextGeneration();
    } finally {
        lock.unlock();
    }
}
```

> 获取等待的线程数量

```java
public int getNumberWaiting() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // parties是总数，count是已经减少的
        // 两者之差就是还在等待的
        return parties - count;
    } finally {
        lock.unlock();
    }
}
```

#### 小结

好了，到这里，基本上关于CyclicBarrier的分析就结束了，从上面的分析中可以看出，CyclicBarrier本身是通过ReentrantLock以及Condition来实现的，本身实现难度不高，但却非常的巧妙

CyclicBarrier的使用也是非常简单的，与CountdownLatch的使用场景有点微妙的区别，使用时千万注意两者的区别

## 总结

好了，到这里，关于四个常用的基于AQS的同步工具的分析就结束了

这篇文章详细地介绍了四个常用的基于AQS的同步工具，包括了其使用的场景，源码的分析，通过源码的分析，我们可以看到，基于AQS的同步器的实现是比较简单的，只需要根据所要实现的同步器的类型(共享模式、独占模式)选择继承的方法`tryXX`、`tryXXShared`就能够实现一个安全的、可靠的同步器了

相信通过这两篇文章的分析，你能更好地理解AQS的设置，已经对应工具的实现原理，在后面开发过程中，能更好地应用这些工具

当然了，关于AQS的工具还远远不止这些，后面有时间的话，我会再写一篇文章，分析其他几个也非常好用的工具，这里先挖个坑