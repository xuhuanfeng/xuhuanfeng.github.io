---
title: LockSupport使用及分析
date: 2019-10-17 19:54:30
tags: 
  - Java
  - LockSupport
  - concurrent
categories:
  - Java
  - Concurrent
---

这篇文章主要是对LockSupport工具进行分析，LockSupport是提供给`concurrent`包中的其他工具使用的底层工具，主要是用于线程的阻塞以及唤醒

<!--more-->

# LockSupport使用及分析

## 介绍

> 以下内容翻译自doc

LockSupport一个基本的线程阻塞工具，主要用于其他同步工具中阻塞线程

每个使用LockSupport的线程会关联一个permit(有且只有一个)，当调用`park`时，如果permit不可用，则可能阻塞，调用`unpark`会是permit重新可用

LockSupport提供高效的，并且没有副作用(与Thread.suspend，Thread.resume对比)，不存在竞态条件，此外，当线程中断或者超时时，也会返回

使用时，需要注意，park方法随时可能返回，所以如果需要检查条件是否满足的情况下，需要将其放在循环中

park方法同时支持一个`blocker`参数，这个对象可以用于记录线程阻塞原因等信息

## 分析

LockSupport大概是`concurrent`包下为数不多代码较少的类了，分析起来也比较容易，直接上代码吧

```java
public class LockSupport {
    // 私有化
    private LockSupport() {}
}
```

从此处也可以看出，LockSupport本身是作为工具使用，因此禁止实例化

核心属性

```java
// unsafe
private static final sun.misc.Unsafe UNSAFE;

private static final long parkBlockerOffset;
private static final long SEED;
private static final long PROBE;
private static final long SECONDARY;

static {
    try {
        // 获取unsafe实例
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        
        Class<?> tk = Thread.class;
        
        // 从thread对象中获取parkBlocker字段
        // doc中说明用于LockSupport
        parkBlockerOffset = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("parkBlocker"));
        
        // 从thread对象中获取threadLocalRandomSeed字段
        // ThreadLocalRandom的当前种子
        SEED = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSeed"));
        
        // 从thread对象中获取threadLocalRandomProbe字段
        // hash值
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
        
        // 从thread对象中获取threadLocalRandomSecondarySeed字段
        // 第二个种子(与ThreadLocalRandom隔离)
        SECONDARY = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

核心方法

```java
private static void setBlocker(Thread t, Object arg) {
    // 通过unsafe为thread的parkBlocker赋值
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
```

```java
// 从t中获取parkBlocker字段的值
public static Object getBlocker(Thread t) {
    if (t == null)
        throw new NullPointerException();
    return UNSAFE.getObjectVolatile(t, parkBlockerOffset);
}
```

```java
// 阻塞当前线程，当permit可用时，才允许线程运行
public static void park() {
    UNSAFE.park(false, 0L);
}

// 参数的blocker
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    // park返回之后，清空对应的blocker
    setBlocker(t, null);
}

// 带时间的park
public static void parkNanos(long nanos) {
    if (nanos > 0)
        UNSAFE.park(false, nanos);
}

// 带时间，带参数
public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, nanos);
        setBlocker(t, null);
    }
}


public static void parkUntil(long deadline) {
    UNSAFE.park(true, deadline);
}

public static void parkUntil(Object blocker, long deadline) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(true, deadline);
    setBlocker(t, null);
}
```

```java
public static void unpark(Thread thread) {
    // 唤醒thread
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

```java
// 获取下一个种子
static final int nextSecondarySeed() {
    int r;
    Thread t = Thread.currentThread();
    if ((r = UNSAFE.getInt(t, SECONDARY)) != 0) {
        r ^= r << 13;   // xorshift
        r ^= r >>> 17;
        r ^= r << 5;
    }
    else if ((r = java.util.concurrent.ThreadLocalRandom.current().nextInt()) == 0)
        r = 1; // avoid zero
    UNSAFE.putInt(t, SECONDARY, r);
    return r;
}
```

可以看到，LockSupport的代码相对来说，是比较简介的，而且从功能的角度来讲，也是比较单一的

## 使用

经过上面的分析之后，对于LockSupport的使用相对来说，就比较简单的，这里给出一个简单的示例

```java
public static void main(String[] args) throws InterruptedException {
    Thread threadA = new Thread(()->{
        Thread currentThread = Thread.currentThread();
        System.out.println(currentThread.getName() + " is working....");
        System.out.println(currentThread.getName() + " is parking...");
        LockSupport.park();
        System.out.println(currentThread.getName() + " is wake up");
    });

    Thread threadB = new Thread(()->{
        Thread currentThread = Thread.currentThread();
        System.out.println(currentThread.getName() + " is working....");
        System.out.println(currentThread.getName() + " waking up threadA");
        LockSupport.unpark(threadA);
        System.out.println(currentThread.getName() + " is finish");
    });

    threadA.start();
    TimeUnit.SECONDS.sleep(1);
    threadB.start();
}
```

输出内容如下

```shell
Thread-0 is working....
Thread-0 is parking...
Thread-1 is working....
Thread-1 waking up threadA
Thread-0 is wake up
Thread-1 is finish
```

从输出的结果可以清楚地看出LockSupport的作用，但需要注意的是，LockSupport推荐在一些偏向于底层的工具中使用，如`ReentrantLock`、`AbstractQueuedSynchronizer`等中使用

在应用中使用，一般是推荐直接使用上一层的工具，而非直接使用LockSupport