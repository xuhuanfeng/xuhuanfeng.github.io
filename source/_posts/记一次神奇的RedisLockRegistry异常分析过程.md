---
title: 记一次神奇的RedisLockRegistry异常分析过程
date: 2020-09-11 15:22:07
tags: 
  - Java
  - Redis
  - 分布式锁
categories:
  - Java
---

# 记一次神奇的RedisLockRegistry异常分析过程

## 问题

早上之前项目组的一个同事说在使用RedisLockRegistry出现了神奇的异常，让我帮忙看下问题所在，他自己分析不出来(其实就是懒，囧....)，刚好早上比较空闲，就看了下问题

异常信息：

```shell
org.springframework.dao.CannotAcquireLockException: Failed to lock mutex at XXX; nested exception is java.nio.BufferOverflowException
	at org.springframework.integration.redis.util.RedisLockRegistry$RedisLock.rethrowAsLockException(RedisLockRegistry.java:362) 
	at org.springframework.integration.redis.util.RedisLockRegistry$RedisLock.lock(RedisLockRegistry.java:356) 
```

```shell
Caused by: java.nio.BufferOverflowException: null
	at java.nio.HeapByteBuffer.put(HeapByteBuffer.java:189) ~[na:1.8.0_121]
	at java.nio.ByteBuffer.put(ByteBuffer.java:859) ~[na:1.8.0_121]
	at org.springframework.integration.redis.util.RedisLockRegistry$LockSerializer.serialize(RedisLockRegistry.java:620) 
	at org.springframework.integration.redis.util.RedisLockRegistry$RedisLock$1.doInRedis(RedisLockRegistry.java:438) 
	at org.springframework.integration.redis.util.RedisLockRegistry$RedisLock$1.doInRedis(RedisLockRegistry.java:423) 
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:207)
```

## 分析

从上面的异常信息来看，应该是在获取锁的时候出现异常`BufferOverflowException`，一开始分析的时候还觉得很奇怪，异常信息提示的是ByteBuffer的问题，好像跟锁本身没啥子挂钩(两个层次的问题，一个是应用层，一个是数据传输层)

从异常信息本身没有获取过多的信息，于是从异常堆栈入手分析，起点是：`RedisLockRegistry.obtainLock#doInRedis`

代码如下：

```java
public Boolean doInRedis(RedisConnection connection) throws DataAccessException {

    long expireAfter = TimeoutUtils.toSeconds(RedisLockRegistry.this.expireAfter,
                                              TimeUnit.MILLISECONDS);
    RedisSerializer<String> serializer = RedisLockRegistry.this.redisTemplate.getStringSerializer();
    byte[][] actualArgs = new byte[][] {
        serializer.serialize(constructLockKey()),
        RedisLockRegistry.this.lockSerializer.serialize(RedisLock.this),
        serializer.serialize("NX"),
        serializer.serialize("EX"),
        serializer.serialize(String.valueOf(expireAfter))
    };

    return connection.execute("SET", actualArgs) != null;
}
```

咋一看好像没啥问题，根据堆栈信息再进一步跟踪：`LockSerializer.serialize`，也就是上面第8行所在的地方，代码如下

```java
private class LockSerializer implements RedisSerializer<RedisLock> {

    @Override
    public byte[] serialize(RedisLock t) throws SerializationException {
        int hostLength = t.lockHost.length;
        int keyLength = t.lockKey.length();
        int threadNameLength = t.threadName.length();
        byte[] value = new byte[1 + hostLength +
                                1 + keyLength +
                                1 + threadNameLength + 8];
        ByteBuffer buff = ByteBuffer.wrap(value);
        buff.put((byte) hostLength)
            .put(t.lockHost)
            .put((byte) keyLength)
            .put(t.lockKey.getBytes())
            .put((byte) threadNameLength)
            .put(t.threadName.getBytes())
            .putLong(t.lockedAt);
        return value;
    }

    @Override
    public RedisLock deserialize(byte[] bytes) throws SerializationException {
        // ....
    }

}
```

这个类的是Spring团队为实现Redis分布式锁实现的一个序列化器，其中序列化和反序列化是对称的，所以挑选其中一个分析，果然，发现了一些端倪

```java
int hostLength = t.lockHost.length;
int keyLength = t.lockKey.length();
int threadNameLength = t.threadName.length();

byte[] value = new byte[1 + hostLength +
                        1 + keyLength +
                        1 + threadNameLength + 8];

ByteBuffer buff = ByteBuffer.wrap(value);

buff.put((byte) hostLength)
    .put(t.lockHost)
    .put((byte) keyLength)
    .put(t.lockKey.getBytes())
    .put((byte) threadNameLength)
    .put(t.threadName.getBytes())
    .putLong(t.lockedAt);
```

这个方面的目的很清晰，就是以：`长度内容长度内容...`这样的形式键锁相关的信息进行序列化

注意1-3行、5-7行、11-17行，这里是有问题的`string.length()`获取的是字符数，比如

```java
String a = "123";
String b = "你好中国";
System.out.println(a.length()); // 3
System.out.println(b.length()); // 4
```

5-7行申请的byte[]长度也是用字符数进行计算的，然而，在11-17行的地方，用的是bytes，也就是字节数，这在单字节字符是正常的，而在多字节字符，比如汉语就有问题了

```java
String a = "123";
String b = "你好中国";
System.out.println(a.getBytes().length); // 3
System.out.println(b.getBytes().length); // 12
```

所以当使用的key或者threadName不是单字节字符时，就会出现文章开始提到的`BufferOverflowException`了，跟同事确认了下，确实线程名用的是英文名，囧.....

这里除了字符长度计算问题外，还有一个隐藏的Bug，同样在11-17行这里，`(byte)XXXX`，这里是将一个int转为byte，所以但int长度超过byte的范围(2^7-1)，即128的时候，就有坑了（溢出）...，放进去是没问题的，但是拿出来的时候，会出现拿一个负数的字节

```java
byte[] host = new byte[buff.get()];
buff.get(host);
// 这里如果是负数的话，就很那啥了...
byte[] lockKey = new byte[buff.get()];
buff.get(lockKey);
```

## 后记

前面提到了，项目中使用的Redis锁版本是：`4.3.9`，这是`SpringBoot 1.5.3.RELEASE`默认的版本，本来还想给官方提个issue，不过翻了下代码记录，在：`5.0.x`分支已经采用其他的方式了，记录及变更内容如下

```shell
commit f7f7bdd067428ae570c6bc0cfe489e1698c0a203
Author: Vedran Pavic <vedran.pavic@gmail.com>
Date:   Fri Mar 31 23:47:27 2017 +0200

INT-4248: Refactor `RedisLockRegistry`

JIRA: https://jira.spring.io/browse/INT-4248

To avoid unexpected double locking behavior in the cluster,
remove the local cache functionality.

Now with the new `clientId` property, the `expire` for the record in
store is always update on each lock operation
```

```java
private boolean obtainLock() {
	Boolean success =
			RedisLockRegistry.this.redisTemplate.execute(
				RedisLockRegistry.this.obtainLockScript,
				Collections.singletonList(this.lockKey), 
				RedisLockRegistry.this.clientId,
				String.valueOf(RedisLockRegistry.this.expireAfter));

	boolean result = Boolean.TRUE.equals(success);

	if (result) {
		this.lockedAt = System.currentTimeMillis();
	}
	return result;
}
```

跟踪上面的execute方法，调用链：`execute-->execute-->keysAndArgs`

最终可以进入到下面的代码中：

```java
protected byte[][] keysAndArgs(RedisSerializer argsSerializer, List<K> keys, Object[] args) {
    final int keySize = keys != null ? keys.size() : 0;
    byte[][] keysAndArgs = new byte[args.length + keySize][];
    int i = 0;
    
    // 序列化key
    if (keys != null) {
        for (K key : keys) {
            if (keySerializer() == null && key instanceof byte[]) {
                keysAndArgs[i++] = (byte[]) key;
            } else {
                keysAndArgs[i++] = keySerializer().serialize(key);
            }
        }
    }
    
    // 序列化参数
    for (Object arg : args) {
        if (argsSerializer == null && arg instanceof byte[]) {
            keysAndArgs[i++] = (byte[]) arg;
        } else {
            keysAndArgs[i++] = argsSerializer.serialize(arg);
        }
    }
    return keysAndArgs;
}
```

可以看到，从5.0.X版本开始，采用的是新的实现方式，其中的`keySerializer()`以及`argsSerializer`，返回的是当前RedisTemplate设定的序列化器，默认是：`JdkSerializationRedisSerializer`

这个就不存在前面提到的问题了

好像又水了一波.....那就这样吧