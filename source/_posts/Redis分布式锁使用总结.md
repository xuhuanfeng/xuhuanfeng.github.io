---
title: Redis分布式锁使用总结
date: 2019-10-19 10:09:55
tags: 
  - Redis
  - 分布式锁
categories:
  - Redis
---

这篇文章主要是对Redis分布式锁进行一个细致的分析，包括Redis分布式锁的核心原理，使用时需要注意的事项，Spring对Redis分布式锁的支持等

<!--more-->

# Redis分布式锁使用总结

## 前言

最近因为项目需要进行多实例的协调，使用到了分布式锁，所以对分布式锁的原理、使用等做了一番调查、学习，顺便将其记录下来，供需要的同学学习交流。

项目中使用的是基于Redis的分布式锁，所以这篇文件的内容都是是基于Redis分布式锁。

## 分布式锁简介

谈起编程语言中的锁，开发者应该是相当熟悉的，当系统中存在多线程并且多线程之间存在竞态条件或者需要协作的时候，我们就会使用到锁，如Java中的`Lock`、`Synchronized`等，但是编程语言中提供的锁，基本上都只适用于在同一个机器上运行的情况，在分布式环境下并不适用。

而在某些情况下，我们是需要在多个机器实例/节点之间进行协作的，这个时候，就需要使用到分布式锁了。

顾名思义，分布式锁就是应用于在分布式环境下多个节点之间进行同步或者协作的锁

分布式锁同普通的锁一样，具有以下几个重要特性

- 互斥性，保证只有持有锁的实例中的某个线程才能进行操作
- 可重入性，同一个实例的同一个线程可以多次获取锁
- 锁超时，支持超时自动释放锁，避免死锁的产生
- 谁加的锁只能由谁释放

## Redis分布式锁原理

由于Redis的命令本身是原子性的，所以，非常适合于作为分布式锁的协调者。

一般情况下，为了保证锁的释放只能由加锁者或者超时释放，一般我们会将对应键的值设置为一个线程唯一标志，如为每个线程生成一个UUID，只有当线程的UUID与锁的值一致时，才能释放锁。

利用Redis来实现分布式的原理非常简单，加锁的时候为某个键设置值，释放的时候将对应的键删除即可。

不过在使用的时候，有一些需要注意的地方，下面我们详细看下基于Redis不同命令来实现分布式锁的操作

### setnx命令

在Redis2.6之前，常用于分布式锁的命令是：`setnx key val`，该命令在对应的键没有值的时候设置成功，存在值的时候设置失败，保证了同时只会有一个连接者设置成功，也即保证同时只会有一个实例的一个线程获取成功。

但是该命令存在一个缺陷，不支持超时机制，所以需要额外的命令来保证能够在超时的情况下释放锁，也就是删除键，可以配合`expire`命令来实现。

由于上述操作涉及到两个命令，所以最好的方式是通过lua脚本来实现加锁的操作，如下所示

```lua
# KEYS[1]是锁的名称，KEYS[2]是锁的值，KEYS[3]是锁的超时时间
local c = redis.call('setnx', KEYS[1], KEYS[2])
if(c == 1) then
   redis.call('expire', KEYS[1], KEYS[3])
end
return c
```

释放锁的时候，需要验证释放锁的是不是锁的持有者，具体代码如下

```lua
# KEYS[1]是锁的名称，KEYS[2]是锁的值
if redis.call('get', KEYS[1]) == KEYS[2] then 
    return redis.call('del', KEYS[1]) 
else return 0 
end
```

### set命令

从上面的setnx命令可以看到，加锁的操作还是比较麻烦的，所以，在Redis2.6之后，redis的set命令进行了增强，设置值的时候，同时支持设置过期时间

```shell
# nx表示不存在的时候设置，ex表示设置过期时间，单位是秒
set LOCK VAL nx ex 15
```

可以看到，通过该命令，进行加锁就方便很多了

释放锁的操作同setnx里提到的释放操作

## Redis分布式锁实现

上面我们提到的是Redis分布式锁的实现原理，不过，每次需要用到锁的时候都需要自己手动实现一次，虽然代码本身没有多少，其实也不是很方便。

正因为如此，有挺多的项目都实现了分布式，并且提供了更加丰富的功能，如下面讨论到的RedisLockRegistry

### RedisLockRegistry

Spring-integration项目是Spring官方提供了集成各种工具的项目，通过integration-redis子项目，提供了非常丰富的功能，关于该项目，后面有时间再写篇文章具体分析一下，这里我们用到其中的一个组件`RedisLockRegistry`

导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-integration</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-redis</artifactId>
</dependency>
```

配置RedisLockRegistry

```java
@Configuration
public class RedisLockConfiguration {

    @Bean
    public RedisLockRegistry redisLockRegistry(
        RedisConnectionFactory redisConnectionFactory) {
        // 注意这里的时间单位是毫秒
        return new RedisLockRegistry(redisConnectionFactory, "registryKey", TIME);
    }
}
```

RedisLockRegistry相当于一个锁的管理仓库，所有的锁都可以从该仓库获取，所有锁的键名为：`registryKey:LOCK_NAME`，默认时间为60s

配置完锁的仓库之后，只需要注入仓库，当需要使用到锁的时候，从仓库中获取一个锁就可以了，如下所示

```java
Lock lock = redisLockRegistry.obtain("redis-lock");
```

该操作返回一个Lock对象，该对象其实是Spring实现的基于Redis的锁，该锁支持了丰富的功能，如`tryLock`等

但使用的时候，只需要跟普通的锁一样操作即可

```java
// lock.tryLock(10, TimeUnit.SECONDS);
lock.lock();
try {
 // ops   
}catch(Exception e) {
    
}finally {
    // 释放锁
    lock.unlock();
}
```

可以看到，通过RedisLockRegistry，我们可以更加方便地使用Redis分布式锁了

### RedisLockRegistry源码分析

上面学习了RedisLockRegistry的使用之后，接下来我们来具体看下RedisLockRegistry的具体实现

![RedisLockRegistry结构](http://image.xuhuanfeng.cn/diagram.png)

从上面的继承结构可以清晰地看出RedisLockRegistry的继承情况，而上面的几个接口基本上都只提供了基本的定义，这里就不展开分析了。直接看RedisLockRegistry的实现

### 构造函数

首先是构造函数，有两个构造函数，如下

```java
private static final long DEFAULT_EXPIRE_AFTER = 60000L;

// 提供了默认的的过期时间，默认过期时间为60s
public RedisLockRegistry(RedisConnectionFactory connectionFactory, String registryKey) {
    this(connectionFactory, registryKey, DEFAULT_EXPIRE_AFTER);
}

public RedisLockRegistry(RedisConnectionFactory connectionFactory, 
                         String registryKey, 
                         long expireAfter) {
    Assert.notNull(connectionFactory, "'connectionFactory' cannot be null");
    Assert.notNull(registryKey, "'registryKey' cannot be null");
    this.redisTemplate = new StringRedisTemplate(connectionFactory);
    this.obtainLockScript = 
        new DefaultRedisScript<>(OBTAIN_LOCK_SCRIPT, Boolean.class);
    this.registryKey = registryKey;
    this.expireAfter = expireAfter;
    this.unlinkAvailable = RedisUtils.isUnlinkAvailable(this.redisTemplate);
}
```

上面第二个构造函数中，有两个没见过的属性，分别是`obtainLockScript`以及`unlinkAvailable`，分析如下

> obtainLockScript

```java
private final RedisScript<Boolean> obtainLockScript;

obtainLockScript = new DefaultRedisScript<>(OBTAIN_LOCK_SCRIPT, Boolean.class);
```

可以看到obtainLockScript是一个DefaultRedisScript实例，该实例的对象用于执行Lua脚本，具体的看下`DefaultRedisScript`的源码

上面的OBTAIN_LOCK_SCRIPT内容如下

```java
private static final String OBTAIN_LOCK_SCRIPT =
			"local lockClientId = redis.call('GET', KEYS[1])\n" +
				"if lockClientId == ARGV[1] then\n" +
					"  redis.call('PEXPIRE', KEYS[1], ARGV[2])\n" +
					"  return true\n" +
					"elseif not lockClientId then\n" +
    				"  redis.call('SET', KEYS[1], ARGV[1], 'PX', ARGV[2])\n" +
					"  return true\n" +
					"end\n" +
					"return false";
```

可以看到，其实就是一段简单的Lua脚本，脚本逻辑如下

1. 调用get命令获取对应的key，如果存在在走2，不存在，则走3
2. 判断key的值是否是入参，如果是，则调用pexire设置过期时间，返回true表示加锁成功
3. 如果不存在，则调用set命令进行加锁，并且设置过期时间，返回true表示加锁成功，从命令中可以看到，使用的参是px，所以构造函数传入的单位是毫秒而不是秒
4. 如果没有执行2、3操作，则返回false，表示加锁失败

> isUnlinkAvailable

该函数检查对应的redis是否支持`UNLINK`命令，该命令用于异步删除某个键，功能等同于`del`命令，但非阻塞，只有在redis4及以上版本才支持

函数内容如下：

```java
public static boolean isUnlinkAvailable(RedisOperations<?, ?> redisOperations) {
  return unlinkAvailable.computeIfAbsent(
		redisOperations, key -> {
			Properties info = redisOperations.execute(
					(RedisCallback<Properties>) connection -> 
					connection.serverCommands().info(SECTION));
			if (info != null) {
				String version = info.getProperty(VERSION_PROPERTY);
				if (StringUtils.hasText(version)) {
					int majorVersion = Integer.parseInt(version.split("\\.")[0]);
					return majorVersion >= 4;
				}
				else {
					return false;
				}
			}
			else {
				throw new IllegalStateException("The INFO command cannot be used in pipeline/transaction.");
			}
		});
	}
```

### 核心

RedisLockRegistry的核心方法其实只有一个，就是obtainLock，具体实现如下

```java
private final Map<String, RedisLock> locks = new ConcurrentHashMap<>();

@Override
public Lock obtain(Object lockKey) {
    Assert.isInstanceOf(String.class, lockKey);
    String path = (String) lockKey;
    return this.locks.computeIfAbsent(path, RedisLock::new);
}
```

可以看到，每一个LockRegistry自己维护了一个`LOCK-KEY`-LOCK的map，这也表明，同一个Registry中，相同的键只会对应一个Lock对象

### RedisLock

从上面的分析中可以看到，LockRegistry维护了一个RedisLock对象的Map，键是锁的名称，值是对应的Lock对象，该对象是Spring实现的一个内部类，具体实现如下所示

#### 构造方法

```java
private RedisLock(String path) {
    this.lockKey = constructLockKey(path);
}
```

RedisLock有且只有一个私有构造方法，所以仅能在当前类中进行构造，这也意味着我们无法自己实例化RedisLock实例

构造的过程非常简单，只是初始化了lockKey，lockKey的内容如下

```java
private String constructLockKey(String path) {
    return RedisLockRegistry.this.registryKey + ":" + path;
}
```

可以看到，lockKey的值其实就是Registry的名称 + `:` + 锁的名称

#### 核心方法

对于一把锁而言，最最核心的方法莫过于加锁和解锁了，RedisLock实现了Lock接口，提供了多样的加锁方式，分别如下所示

##### 不可中断锁

```java
private final ReentrantLock localLock = new ReentrantLock();

@Override
public void lock() {
    this.localLock.lock();
    while (true) {
        try {
            while (!obtainLock()) {
                Thread.sleep(100); //NOSONAR
            }
            break;
        }
        catch (InterruptedException e) {
            // 不可中断，所以忽略中断异常
        }
        catch (Exception e) {
            this.localLock.unlock();
            rethrowAsLockException(e);
        }
    }
}
```

从上面的代码可以看到，lock方法首先尝试获取`ReentrantLock`，如果获取成功，才尝试去获取分布式锁，获取`localLock`的目的在于，如果本地有多个线程在竞争该锁，则只有获取到本地的锁的线程才能尝试去获取分布式锁，好处在于，减少了不必要的网络开销，提高性能

由于lock方法明确规定，如果获取不到锁，则进行阻塞，直至获取到锁或者出现异常，所以上面每隔100毫秒会去尝试获取锁，直到获取成功或者抛出异常为止

获取锁的代码也非常简单，如下所示

```java
// 实例化Registry的时候进行初始化
private final String clientId = UUID.randomUUID().toString();

private boolean obtainLock() {
    Boolean success =
      RedisLockRegistry.this.redisTemplate.execute(
        // 获取锁的lua脚本
        RedisLockRegistry.this.obtainLockScript,
        // 获取的锁名称
        Collections.singletonList(this.lockKey), 
        // 锁的内容
        RedisLockRegistry.this.clientId,
        // 锁的过期时间
        String.valueOf(RedisLockRegistry.this.expireAfter));

    boolean result = Boolean.TRUE.equals(success);
	
    // 如果获取成功，则记录锁的时间
    if (result) {
        this.lockedAt = System.currentTimeMillis();
    }
    return result;
}
```

从上面获取锁的代码可以看到，每一个LockRegistry实例只会有一个值，该值在Registry实例化的时候通过UUID生成，一个实例内的多个线程之间的竞争直接通过ReentrantLock进行，不涉及到Redis相关的操作。

##### 可中断锁

```java
@Override
public void lockInterruptibly() throws InterruptedException {
    this.localLock.lockInterruptibly();
    try {
        while (!obtainLock()) {
            Thread.sleep(100); //NOSONAR
        }
    }
    catch (InterruptedException ie) {
        // 释放锁，并且响应中断信号
        this.localLock.unlock();
        Thread.currentThread().interrupt();
        throw ie;
    }
    catch (Exception e) {
        this.localLock.unlock();
        rethrowAsLockException(e);
    }
}
```

看懂了lock的代码，再来看lockInterruptibly就非常简单了，lock不响应中断信号，则lockInterruptibly则相应中断信号，因此，获取锁的过程如果出现中断，则结束获取操作了

##### 尝试获取锁

尝试获取锁以为着如果能获取锁，则获取，如果不能获取，则结束，当然，可以附带等待是时间，有两个版本的tryLock，如下

```java
@Override
public boolean tryLock() {
    try {
        // 调用另一个tryLock，并且将时间设置为0
        return tryLock(0, TimeUnit.MILLISECONDS);
    }
    catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return false;
    }
}

@Override
public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
    long now = System.currentTimeMillis();
    // 先尝试获取本地锁，如果在指定时间内无法获取到本地锁，则放弃
    if (!this.localLock.tryLock(time, unit)) {
        return false;
    }
    try {
        
        // 记录获取锁到期时间
        long expire = now + TimeUnit.MILLISECONDS.convert(time, unit);
        boolean acquired;
        
        // 如果获取不到锁，并且时间还有剩余，则先休眠100毫秒，然后继续尝试获取
        while (!(acquired = obtainLock()) && System.currentTimeMillis() < expire) { 
            Thread.sleep(100); //NOSONAR
        }
        // 到这里表示获取锁超时
        // 如果无法获取到分布式锁，则释放本地锁
        if (!acquired) {
            this.localLock.unlock();
        }
        return acquired;
    }
    catch (Exception e) {
        this.localLock.unlock();
        rethrowAsLockException(e);
    }
    return false;
}
```

具体的分析都详细写在注释里了，补充一点就是，从tryLock的实现中可以看到，tryLock本身是响应中断的，与接口的定义一致

##### 释放锁

```java
// 判断锁的所有者是否是当前实例
public boolean isAcquiredInThisProcess() {
    return RedisLockRegistry.this.clientId.equals(
        RedisLockRegistry.this.redisTemplate.boundValueOps(this.lockKey).get());
}

// 删除对应的键，也即释放分布式锁
private void removeLockKey() {
    if (this.unlinkAvailable) {
        RedisLockRegistry.this.redisTemplate.unlink(this.lockKey);
    }
    else {
        RedisLockRegistry.this.redisTemplate.delete(this.lockKey);
    }
}

@Override
public void unlock() {
    // 如果尝试释放的不是本线程加的锁，则抛出异常
    if (!this.localLock.isHeldByCurrentThread()) {
        throw new IllegalStateException("You do not own lock at " + this.lockKey);
    }
    // 当前线程持有的锁的数量，即重入的次数
    // 如果此时 > 1，表示当前线程有多次获取锁，释放的时候只减少本地锁的次数
    // 此时其他的方法还持有锁，不能释放分布式锁
    if (this.localLock.getHoldCount() > 1) {
        this.localLock.unlock();
        return;
    }
    try {
        // 此时分布式锁已经由于超时被释放了，抛出异常
        if (!isAcquiredInThisProcess()) {
            throw new IllegalStateException("Lock was released in the store due to expiration. " + "The integrity of data protected by this lock may have been compromised.");
        }
		
        // 如果收到中断信号，则异步释放锁
        // 尽快响应中断...
        if (Thread.currentThread().isInterrupted()) {
            RedisLockRegistry.this.executor.execute(this::removeLockKey);
        }
        else {
            removeLockKey();
        }

        if (logger.isDebugEnabled()) {
            logger.debug("Released lock; " + this);
        }
    }
    catch (Exception e) {
        ReflectionUtils.rethrowRuntimeException(e);
    }
    finally {
        this.localLock.unlock();
    }
}
```

## 总结

本文主要简单介绍了分布式锁，在Redis中使用分布式锁的原理，本质就是`set`或者`setnx`命令的使用，以及对应版本的加锁以及解锁操作。

最后分析了RedisLockRegistry的具体实现，RedisLockRegistry是Spring提供的基于Redis的分布式锁的实现，主要包含两部分，一部分是本地锁，用于一个实例下多个线程的协调，只有获取到本地锁的线程才去尝试获取分布式锁，通过这种方式来提高获取锁的性能

