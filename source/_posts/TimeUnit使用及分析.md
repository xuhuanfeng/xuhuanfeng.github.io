---
title: TimeUnit使用及分析
date: 2019-10-06 14:35:10
tags: 
  - Java
  - Concurrent
  - TimeUnit
categories:
  - Java
  - Concurrent
---

这篇文章主要是学习`TimeUnit`过程的一个记录及分析

TimeUnit是`java.util.concurrent`包下一个非常有用而且非常好用的工具，用于方便地进行各种时间单位的转换以及提供一些多时间单位的工具

<!--more-->

# TimeUnit使用及分析

## 简介

>  以下内容来自doc

TimeUnit代表在给定单位内的连续时间，并且提供了一系列的工具用于进行单位转换，计时、延迟等操作

TimeUnit本身不维护时间信息，仅对时间操作

## 使用

先看下TimeUnit的使用，然后后面再分析下其具体的实现

### 时间转换

在没有TimeUnit之前，时间转换是一个比较麻烦的事情，每次都需要人工计算，如将mill转分钟，有了该工具，就非常方便，而且不容易出错

```java
public static void testTransform() {
    long now = System.nanoTime();
    System.out.println("Now in nano: " + now);
    System.out.println("Now in micro: " + now / 1000);
    System.out.println("Now in mill: " + now / 1000 / 1000);
    System.out.println("Now in second: " + now / 1000 / 1000 / 1000);
    System.out.println("Now in minute: " + now / 1000 / 1000 / 1000 / 60);

    System.out.println("TimeUnit=========");

    System.out.println("Now in nano: " + now);
    System.out.println("Now in micro: " + TimeUnit.NANOSECONDS.toMicros(now));
    System.out.println("Now in mill: " + TimeUnit.NANOSECONDS.toMillis(now));
    System.out.println("Now in second: " + TimeUnit.NANOSECONDS.toSeconds(now));
    System.out.println("Now in minute: " + TimeUnit.NANOSECONDS.toMinutes(now));
}
```

### 任意时间单位的等待

JDK默认提供的wait方法仅支持milliseconds作为时间单位，显然，如果需要其他单位，进行转换也是个麻烦的事情

所以TimeUnit也提供了方便的方式

```java
public synchronized void testWait() {
    // 等待一分钟
    System.out.println(new Date());
    try {
        // 人工计算，容易出错
        wait(1000 * 60);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println(new Date());

    System.out.println("TimeUnit=========");

    System.out.println(new Date());
    try {
        TimeUnit.MINUTES.timedWait(this, 1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println(new Date());
}
```

### 任意时间单位休眠

有时候，需要调用Thread.sleep()，但是该方法同样仅支持毫秒为单位

```java
public void testSleep() {
    // sleep一分钟
    try {
        Thread.sleep(1000 * 60);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println("TimeUnit=========");

    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

还有一个timedJoin也是对Thread.join的增强，就不举例了

### 小结

TimeUnit本身并没有引入新的概念，只是将一些原本比较繁琐，容易出错的时间单位之间的转换操作封装起来，提供更加简便的接口而已

## 分析

TimeUnit本身是一个枚举类

包含了若干个实例，对应不同的时间单位

```java
public enum TimeUnit {
    /**
     * 纳秒
     */
    NANOSECONDS {},

    /**
     * 微秒
     */
    MICROSECONDS {},

    /**
     * 毫秒
     */
    MILLISECONDS {},

    /**
     * 秒
     */
    SECONDS {},

    /**
     * 分钟
     */
    MINUTES {},

    /**
     * 小时
     */
    HOURS {},

    /**
     * 天
     */
    DAYS {};
}
```

同时，为了更好地进行单位转换，提供了一系列的方法，默认情况下这些方法都抛出异常，具体的内容由其实例(子类)进行实现

```java
// 将给定时间转换为指定单位对应的时间
public long convert(long sourceDuration, TimeUnit sourceUnit) {
    throw new AbstractMethodError();
}

// 转为纳秒
public long toNanos(long duration) {
    throw new AbstractMethodError();
}

// 转为微秒
public long toMicros(long duration) {
    throw new AbstractMethodError();
}

// 转为毫秒
public long toMillis(long duration) {
    throw new AbstractMethodError();
}

// 转为秒
public long toSeconds(long duration) {
    throw new AbstractMethodError();
}

// 转为分钟
public long toMinutes(long duration) {
    throw new AbstractMethodError();
}

// 转为小时
public long toHours(long duration) {
    throw new AbstractMethodError();
}

// 转为天
public long toDays(long duration) {
    throw new AbstractMethodError();
}
```

大概所有实例都会使用到的一个转换方法

```java
// 计算 d * m，同时反之溢出
static long x(long d, long m, long over) {
    if (d >  over) return Long.MAX_VALUE;
    if (d < -over) return Long.MIN_VALUE;
    return d * m;
}
```

此外，为了换算的方便，还提供了几个静态变量(分别对应上述的各个时间之间的转换关系)

```java
static final long C0 = 1L;
static final long C1 = C0 * 1000L;
static final long C2 = C1 * 1000L;
static final long C3 = C2 * 1000L;
static final long C4 = C3 * 60L;
static final long C5 = C4 * 60L;
static final long C6 = C5 * 24L;
```

由于这些转换相对来说比较简单，这里只分析其中一两个具体实现

```java
NANOSECONDS {
    // nano转nano，直接返回即可
    public long toNanos(long d)   { return d; }
    // nano转micro，c1/c0算出mico比nano的倍数，d/倍数，换算成micro
    public long toMicros(long d)  { return d/(C1/C0); }
    // 同上，不展开了
    public long toMillis(long d)  { return d/(C2/C0); }
    public long toSeconds(long d) { return d/(C3/C0); }
    public long toMinutes(long d) { return d/(C4/C0); }
    public long toHours(long d)   { return d/(C5/C0); }
    public long toDays(long d)    { return d/(C6/C0); }
    public long convert(long d, TimeUnit u) { return u.toNanos(d); }
    // 注意这个方法，是TimeUnit的一个抽象方法，用于计算d比m多出来的微秒数（除了nano、micro外其他都为0）
    int excessNanos(long d, long m) { return (int)(d - (m*C2)); }
}
```

```java
DAYS {
    // 通过x函数计算，同时避免溢出，最大允许转换为Long.MAX_VALUE/一天的nano数
    public long toNanos(long d)   { return x(d, C6/C0, MAX/(C6/C0)); }
    public long toMicros(long d)  { return x(d, C6/C1, MAX/(C6/C1)); }
    public long toMillis(long d)  { return x(d, C6/C2, MAX/(C6/C2)); }
    public long toSeconds(long d) { return x(d, C6/C3, MAX/(C6/C3)); }
    public long toMinutes(long d) { return x(d, C6/C4, MAX/(C6/C4)); }
    public long toHours(long d)   { return x(d, C6/C5, MAX/(C6/C5)); }
    public long toDays(long d)    { return d; }
    public long convert(long d, TimeUnit u) { return u.toDays(d); }
    int excessNanos(long d, long m) { return 0; }
};
```

另外几个比较常用的方法

```java
// 支持多种时间单位的wait
public void timedWait(Object obj, long timeout)
    throws InterruptedException {
    if (timeout > 0) {
        // 先转mill
        long ms = toMillis(timeout);
        // 计算剩余的nano
        int ns = excessNanos(timeout, ms);
        obj.wait(ms, ns);
    }
}
```

```java
// 支持多种时间单位的join
public void timedJoin(Thread thread, long timeout)
    throws InterruptedException {
    if (timeout > 0) {
        long ms = toMillis(timeout);
        int ns = excessNanos(timeout, ms);
        thread.join(ms, ns);
    }
}
```

```java
// 支持多种时间单位的sleep，大概是TimeUnit中用得最多的方法了
public void sleep(long timeout) throws InterruptedException {
    if (timeout > 0) {
        long ms = toMillis(timeout);
        int ns = excessNanos(timeout, ms);
        Thread.sleep(ms, ns);
    }
}
```