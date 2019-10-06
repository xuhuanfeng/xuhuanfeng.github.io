---
title: AtomicXXX使用及分析
date: 2019-10-06 14:29:41
tags: 
  - Java
  - Concurrent
  - Atomic
  - CAS
categories: 
  - Java
  - Concurrent
---

这篇文章主要是学习`java.util.concurrent.atomic`包过程中的记录及分析

atomic包下提供了众多的无锁的，线程安全的并发工具，在某些程度上扩充了Java的并发体系

<!--more-->

# AtomicXXX使用及分析

> 接下来的内容主要针对java.util.concurrent.atomic进行学习和分析

## 背景

正如我们所知道的，Java中的递增操作，如`i++`并不是原子操作，也即是非线程安全，具体原因可以通过下面的例子看出

```java
public class IntegerTest {

    private int i = 0;

    public void incr() {
        i++;
    }

    public static void main(String[] args) {
        new IntegerTest().incr();
    }
}
```

这个例子包含一段非常简单代码，定义一个int变量，然后在方法中调用递增操作

下面的代码是通过`javap -v XXX`打印出来的字节码信息(仅incr方法)

```java
public void incr();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lcn/xuhuanfeng/concurrency/atomic/primary/IntegerTest;

```

这里有一个地方需要注意，不能使用局部变量，如下面的例子

```java
public void incr() {
    int i = 0;
    i++;
}
```

此时由于i被封闭在了线程的栈帧中(一个方法对应一个栈帧)，在该栈帧中，变量是线程安全的，所以对应的操作也是安全的，javac直接采用iinc指令进行操作

```java
  public void incr();
    Code:
       0: iconst_0
       1: istore_1
       2: iinc          1, 1 // 直接调用iinc对局部变量进行递增操作
       5: return
```

回到上面例子中的字节码

```java
public void incr();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0 // 将this加载到操作数中，0号局部变量为this(下面的LocalVariableTable)
         1: dup // 复制一份栈顶数据，即this
         2: getfield      #2                  // Field i:I，获取字段i的值，并且push到操作数栈中
         5: iconst_1     // 将 1 push到操作数栈中
         6: iadd         // 将 栈顶两个元素，即 0 和 1 进行相加
         7: putfield      #2                  // Field i:I 将字段i写入到对象中
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lcn/xuhuanfeng/concurrency/atomic/primary/IntegerTest;
```

从字节码中可以看出，操作对象属性的时候，是先将其读取到操作数栈中，然后执行操作，再将其写入，也就是包含三个部分的操作：获取，操作，写入，这也印证了我们上面提到的i++本身并非原子操作

回到本节的主题，正是由于i++本身不是线程安全的，所以，在并发环境下进行此类操作的时候，需要进行同步处理，传统的做法是对方法加监视器锁，也即`synchronized`关键字或者加锁，即`Lock`接口的实现，这两种方式都是不错的方式，但，这两种锁本身都会带来一定的开销，如获取锁，释放锁等操作

在JDK1.5中，引入了`java.util.concurrent.atomic`包，并且提供了一系列的原子操作工具，如`AtomicInteger`、`AtomicLong`等，用于作为锁之外的另一种处理方式

atomic包中的所有操作都是基于CAS(Compare And Swap)机制进行处理的

## CAS

CAS，全称是Compare And Swap，是一种基于无锁机制，需要CPU本身的支持，目前来说，使用的是CPU提供的指令`CMPXCHG `，关于该指令，可以参考[cmpxchg](<https://www.felixcloutier.com/x86/cmpxchg>)，不过我没看懂，囧.....

atomic包中的操作都是基于CAS进行的

正如上面提及的，CAS需要CPU的支持，也即进行CAS操作的时候，需要使用原生的指令，而不是JVM指令，那么问题来了，在Java中是如何实现的呢

通过AtomicInteger中的代码我们可以大致了解一下

```java
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

可以看到，AtomicInteger中的cas操作是通过unsafe对象来操作的

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
```

跟踪进去Unsafe类

```java
package sun.misc;
public final class Unsafe {
    private static final Unsafe theUnsafe;
    
    @CallerSensitive
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();
        // 检查是否是由Bootstrap Classloader
        if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }
    
    static {
        registerNatives();
        Reflection.registerMethodsToFilter(Unsafe.class, new String[]{"getUnsafe"});
        theUnsafe = new Unsafe();
    }
}
```

可以看到，Unsafe类的包名是`sun.misc`，并且Unsafe类仅能由Bootstrap Classloader进行加载，所以我们无法直接获取，但是也可以有其他的方法来操作，具体可以参考美团团队的文章：[Java魔法类：Unsafe应用解析](<https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html>)

跟踪进去之后，可以看到方法的签名：`public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);`

由于是native方法，这里我就不展开了(其实主要是没看懂)

虽然看不懂，但也不影响对CAS机制的理解，CAS机制的原理也非常简单，其假设大部分情况下是没有人跟我们争数据的(乐观)，所以，不需要在每次修改的时候都获取锁，操作，再解锁，而是**每次操作之前，先比较一下，看一下内存中的值是不是期待的值，如果是，则将其修改为目标值，否则，则说明有人在我们之前修改的数据，导致数据与预期的不一致，所以本次操作失败**，当然，这个操作本身也不是原子的，但是有CPU的支持，一切就好说了

需要注意的是，CAS本身只会进行一次，如果失败了，那就失败了，但是这种操作给了我们一种启发，既然一次失败，那不停重试，直到成功，这种机制也称为自旋，即本身不拿锁，不阻塞，一直尝试，直到成功

可以看下Unsafe中的一个实现

```java
// atomicInteger中的incrementAndGet就是直接调用该方法来实现的
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

## AtomicXXX

知道了Atomic本身是基于CAS的，并且大致了解了CAS的原理之后，关于AtomicXXX的使用其实就很简单了，每次操作的时候，都Atomic都是调用CAS，当需要一定设置成功的时候，就采用自旋机制进行操作

可以将AtomicXXX理解为对应的XXX的包装，并且提供了无锁的，能够保证线程安全的操作

需要注意的是，AtomicXXX中的对应字段的值都是volatile类型的，目的是为了保证可加性

```java
private volatile int value;
private volatile long value;
private volatile V value;
```

### AtomicInteger

```java
public void testAtomicInteger() {
    AtomicInteger atomicInteger = new AtomicInteger(4);

    System.out.println(atomicInteger.get());

    // 自增并且返回自增之后的值， ++i
    System.out.println(atomicInteger.incrementAndGet());

    // 先返回再自增， i++
    System.out.println(atomicInteger.getAndIncrement());

    // i += n
    System.out.println(atomicInteger.getAndAdd(10));

    // --i
    System.out.println(atomicInteger.decrementAndGet());

    // i--
    System.out.println(atomicInteger.getAndDecrement());

    // i = 4
    atomicInteger.set(4);

    // 如果内存中的值为4，则更新为10
    boolean result = atomicInteger.compareAndSet(4, 10);
}
```

对于AtomicInteger，基本上常用的也就这几个方法，更多的内容参考文档即可

### AtomicBoolean

相比于AtomicInteger，AtomicBoolean的操作就更加简单了

AtomicBoolean本身并没有直接对布尔值进行操作(主要是unsafe中也没有提供相应的操作)，本身实际上还是对int进行操作，1为true，0为false

```java
public final boolean compareAndSet(boolean expect, boolean update) {
    int e = expect ? 1 : 0;
    int u = update ? 1 : 0;
    return unsafe.compareAndSwapInt(this, valueOffset, e, u);
}
```

看到这里，关于AtomicBoolean的操作就自然懂了，由目标是布尔类型，所以AtomicBoolean本身支持的操作不多

### AtomicLong

基本上AtomicLong提供的方法跟AtomicInteger类似，只是底层使用的unsafe提供的long相关的操作，所以这里就不展开了，参考下下面的方法签名即可

```java
public class AtomicLong {
  public AtomicLong(long);
  public AtomicLong();
  public final long get();
  public final void set(long);
  public final void lazySet(long);
  public final long getAndSet(long);
  public final boolean compareAndSet(long, long);
  public final boolean weakCompareAndSet(long, long);
  public final long getAndIncrement();
  public final long getAndDecrement();
  public final long getAndAdd(long);
  public final long incrementAndGet();
  public final long decrementAndGet();
  public final long addAndGet(long);
  public jString toString();
  public int intValue();
  public long longValue();
  public float floatValue();
  public double doubleValue();
}
```

### AtomicReference

AtomicReference的使用也基本同上，只是操作的目标是对象，而非原始数据类型，底层使用的是unsafe提供的关于对象相关的操作如：`compareAndSwapObject`、`getAndSetObject`等

方法签名如下

```java
public class AtomicReference<V>  {
  public AtomicReference(V);
  public AtomicReference();
  public final V get();
  public final void set(V);
  public final void lazySet(V);
  public final boolean compareAndSet(V, V);
  public final boolean weakCompareAndSet(V, V);
  public final V getAndSet(V);
  public String toString();
}
```

### AtomicIntegerArray

atomic包中，除了提供对于对象(原始类型，Object类型)相关的操作外，还对数组(特殊对象)提供相关的操作，当然，还是借助unsafe包中提供的工具的支持

数组，实际上就是一大块连续的内存区域，存储着相同数据类型的数据，所以，对数组的操作，实际上就是对这一块区域内存中某个部分进行操作，了解过c或者c++的同学应该比较清楚这一点，通过基址地址(数组头)+偏移即可操作数据中的每个元素

下面详细分析下AtomicIntegerArray的具体实现

属性

```java
// 对应的unsafe操作
private static final Unsafe unsafe = Unsafe.getUnsafe();
// 获取基于数据的类中数据的第一个元素地址
// 注意，这里不是获取某个数组对象的地址，是获取数组数据在对象中的相对位置
// 如，int数组中的数据是在偏移量16(win,jdk64)开始，这个跟哪个数据对象是无关的
private static final int base = unsafe.arrayBaseOffset(int[].class);
// 元素大小的位数
private static final int shift;
// 实际的元素所存储的位置，注意是final类型
private final int[] array;
```

初始化

```java
static {
    // 获取每个元素的大小，如int[] 为4
    int scale = unsafe.arrayIndexScale(int[].class);
    // 检查元素大小是否是2的倍数，不是则报错
    if ((scale & (scale - 1)) != 0)
        throw new Error("data type scale not a power of two");
    // 31-scale前置的0个数，也就是获取到元素大小所占的位数，如：int[] => 31 - 29 = 2
    shift = 31 - Integer.numberOfLeadingZeros(scale);
}
```

辅助方法

```java
// 检查下标是否合法，由于是直接操作内存，避开了VM的下标检查，所以这一步是必须的
private long checkedByteOffset(int i) {
    if (i < 0 || i >= array.length)
        throw new IndexOutOfBoundsException("index " + i);

    return byteOffset(i);
}

// 获取该下标对应的地址，base + i * 2^size
private static long byteOffset(int i) {
    return ((long) i << shift) + base;
}

// 直接通过unsafe获取位置的数据
private int getRaw(long offset) {
    return unsafe.getIntVolatile(array, offset);
}
```

构造方法

```java
// final类型保证赋值的时候的可见性，所以这里不存在可见性问题
public AtomicIntegerArray(int length) {
    array = new int[length];
}

public AtomicIntegerArray(int[] array) {
    this.array = array.clone();
}
```

操作方法

```java
// 获取长度 
public final int length() {
     return array.length;
 }
```

```java
// 获取某个元素
public final int get(int i) {
    return getRaw(checkedByteOffset(i));
}
```

```java
// 设置某个元素
public final void set(int i, int newValue) {
    unsafe.putIntVolatile(array, checkedByteOffset(i), newValue);
}
```

```java
// 获取并且返回
public final int getAndSet(int i, int newValue) {
    return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
}
```

```java
// 比较并设置
public final boolean compareAndSet(int i, int expect, int update) {
    return compareAndSetRaw(checkedByteOffset(i), expect, update);
}

private boolean compareAndSetRaw(long offset, int expect, int update) {
    return unsafe.compareAndSwapInt(array, offset, expect, update);
}
```

其他的方法使用起来大致类似，就不展开了，可以看到，对数组的操作，本质其实也是对某个元素的操作

### AtomicLongArrray

分析完AtomicIntegerArray之后，AtomicLongArray就不成问题了，这里就不展开了，基本上思路是一样的，只是借助的unsafe方法不同

### AtomicReferenceArray

同上，只是数据的内容改为Object而已

### AtomicXXXFieldUpdater

在atomic包中，还存在一种特殊类型的工具，同样对应三种基本类型

- AtomicIntegerFieldUpdater
- AtomicLongFieldUpdater
- AtomicReferenceFieldUpdater

通过其名称可以看出，这几个工具是用于更新字段的(FieldUpdater)

以下内容来自doc

> 一个基于反射的工具，用于原子性更新指定类的指定的volatile字段(int、long、reference)的值

从doc中就可以清晰地看出了，这几个工具类用于原子更新某个类的某个字段，这些字段必须是volatile类型的(保证可见性)

之所以需要这几个工具，是有时候我们已经设计好了某个类，或者由于历史原因，无法更改该类的某些字段，在并发环境下操作的时候除了直接加锁外，也可以使用这几个工具来轻化操作的代价

下面以AtomicIntegerFieldUpdater为例进行分析

```java
public abstract class AtomicIntegerFieldUpdater<T> {}
```

从类声明可以看到，该类是一个抽象类，并且是泛型的，其中的泛型代表的就是目标类的类型了

核心方法

```java
// 构造器为protected，仅能继承使用
protected AtomicIntegerFieldUpdater();

// 工厂方法，参数为：执行类型和所要操作的字段
public static <U> AtomicIntegerFieldUpdater<U> newUpdater(Class<U>, String);

// 剩下几个方法都是比较熟悉的了，都是对指定字段的操作而已，同AtomicXXX
public abstract boolean compareAndSet(T, int, int);
public abstract boolean weakCompareAndSet(T, int, int);
public abstract void set(T, int);
public abstract void lazySet(T, int);
public abstract int get(T);
public int getAndSet(T, int);
public int getAndIncrement(T);
public int getAndDecrement(T);
public int getAndAdd(T, int);
public int incrementAndGet(T);
public int decrementAndGet(T);
public int addAndGet(T, int);
```

在AtomicIntegerFiledUpdater中，提供了一个私有内部实现

```java
private static final class AtomicIntegerFieldUpdaterImpl<T>
        extends AtomicIntegerFieldUpdater<T> {}
```

下面着重分析该类

核心字段

```java
// 本身还是通过unsafe来实现cas
private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();

// 字段所在的偏移位置
private final long offset;

private final Class<?> cclass;

// 字段所在的类
private final Class<T> tclass;
```

构造器

```java
// 参数1：目标类的类型
// 参数2：目标字段
// 参数3：调用者类型，由于Java体系存在访问控制，所以需要鉴别调用者是否有权限操作该字段
AtomicIntegerFieldUpdaterImpl(final Class<T> tclass,
                              final String fieldName,
                              final Class<?> caller) {
    final Field field;
    final int modifiers;
    try {
        // 获取指定类的指定字段
        field = AccessController.doPrivileged(
            new PrivilegedExceptionAction<Field>() {
                public Field run() throws NoSuchFieldException {
                    return tclass.getDeclaredField(fieldName);
                }
            });
        
        // 确保调用者有权限访问
        modifiers = field.getModifiers();
        sun.reflect.misc.ReflectUtil.ensureMemberAccess(
            caller, tclass, null, modifiers);
        
        ClassLoader cl = tclass.getClassLoader();
        ClassLoader ccl = caller.getClassLoader();
        
        // 如果类加载器为null，则为Bootstrap classLoader
        // 如果调用者不是有bootstrap加载，并且两者不是由同一个类加载器加载
        // 并且目标类是由bootstrap加载或者两个加载器之间不是父子关系
        // 则检查是否具有该包的访问权限
        if ((ccl != null) && (ccl != cl) &&
            ((cl == null) || !isAncestor(cl, ccl))) {
            sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
        }
     // 没有权限访问，直接抛出异常
    } catch (PrivilegedActionException pae) {
        throw new RuntimeException(pae.getException());
    // 其他类型异常
    } catch (Exception ex) {
        throw new RuntimeException(ex);
    }

    // 目标字段只能是int类型
    if (field.getType() != int.class)
        throw new IllegalArgumentException("Must be integer type");

    // 目标字段只能是volatile类型
    if (!Modifier.isVolatile(modifiers))
        throw new IllegalArgumentException("Must be volatile type");

    // 如果是protect类型并且两者存在继承关系，并且不在同一个包
    // 则转为子类
    this.cclass = (Modifier.isProtected(modifiers) &&
                   tclass.isAssignableFrom(caller) &&
                   !isSamePackage(tclass, caller))
        ? caller : tclass;
    this.tclass = tclass;
    this.offset = U.objectFieldOffset(field);
}

// 判断是否是同一个包，直接通过包名
private static boolean isSamePackage(Class<?> class1, Class<?> class2) {
    return class1.getClassLoader() == class2.getClassLoader()
        && Objects.equals(getPackageName(class1), getPackageName(class2));
}

private static String getPackageName(Class<?> cls) {
    String cn = cls.getName();
    int dot = cn.lastIndexOf('.');
    return (dot != -1) ? cn.substring(0, dot) : "";
}
```

核心操作（仅分析其中一个，其他的类似啦）

```java
public final boolean compareAndSet(T obj, int expect, int update) {
    accessCheck(obj);
    return U.compareAndSwapInt(obj, offset, expect, update);
}

private final void accessCheck(T obj) {
    // 检查obj是否是传入的目标类的实例
    if (!cclass.isInstance(obj))
        throwAccessCheckException(obj);
}

private final void throwAccessCheckException(T obj) {
    if (cclass == tclass)
        throw new ClassCastException();
    else
        throw new RuntimeException(
        new IllegalAccessException(
            "Class " +
            cclass.getName() +
            " can not access a protected member of class " +
            tclass.getName() +
            " using an instance of " +
            obj.getClass().getName()));
}
```

至此可以看到，AtomicXXXFieldUpdater同样也是通过CAS来更新指定字段，只是多做了一些访问的限制，检查

AtomicLongFiledUpdater和AtomicReferenceFieldUpdater基本也类似，就不展开了

在使用该工具的时候，需要注意

- 调用者必须要有权限访问目标类的字段，如目标类的字段为private时，是不可访问的
- 字段必须是volatile类型的
- 字段必须是实例类型，不能是static修饰
- 字段不能是final类型

### AtomicStampedReference

在CAS的应用过程中，人们发现其存在一个问题，称之为ABA问题，示例如下

1. 线程1，执行CAS(A,B)，然后执行CAS(B,A)

1. 线程2，执行CAS(A,B)，此时CAS依旧成功，但是A已经不再是之前的A(虽然数值是一样的)

一般会通过版本号之类的机制来解决ABA问题，每次操作记录更新版本号，比较的时候带上版本号进行比较

在JDK中提供了AtomicStampedReference来处理这种情况

```java
public class AtomicStampedReference<V> {}
```

下面详细分析

核心字段

```java
private volatile Pair<V> pair;

private static class Pair<T> {
    final T reference;
    final int stamp;
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}
```

可以看到，AtomicStampedReference中维护了一个pair属性，该属性是Pair的一个实例，Pair的定义也非常简单，就是一个引用加一个版本号

构造器

```java
// 使用提供的引用及版本号来初始化pair
public AtomicStampedReference(V initialRef, int initialStamp) {
    pair = Pair.of(initialRef, initialStamp);
}
```

核心方法

```java
public V getReference() {
    return pair.reference;
}

public int getStamp() {
    return pair.stamp;
}

public V get(int[] stampHolder) {
    Pair<V> pair = this.pair;
    // 版本号存放在stampHolder[0]
    stampHolder[0] = pair.stamp;
    return pair.reference;
}
```

```java
// cas，参数多了版本号信息
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        // 比较引用值及版本号
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        // 新值与当前值一样，或者更新成功
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}

// 直接通过unsafe进行更新
private boolean casPair(Pair<V> cmp, Pair<V> val) {
        return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}
```

```java
//set 
public void set(V newReference, int newStamp) {
    Pair<V> current = pair;
    if (newReference != current.reference || newStamp != current.stamp)
        this.pair = Pair.of(newReference, newStamp);
}
```

```java
// just update stamp
public boolean attemptStamp(V expectedReference, int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        (newStamp == current.stamp ||
         casPair(current, Pair.of(expectedReference, newStamp)));
}
```

AtomicStampedReference提供的方法总体上比其他的少得多，常用的也就是compareAndSet了，通过代码的分析，也可以看到，其实就是在操作之前先比较一下引用的数据以及版本号，如果与预期一致，则采用CAS进行更新

### AtomicMarkableReference

前面提到了AtomicStrampedReference，从代码中可以看到，使用的是一个int作为版本号，而AtomicMarkableReference则可以视为其简化版本，使用的是boolean作为标记号

```java
private static class Pair<T> {
    final T reference;
    // 注意这里
    final boolean mark;
    private Pair(T reference, boolean mark) {
        this.reference = reference;
        this.mark = mark;
    }
    static <T> Pair<T> of(T reference, boolean mark) {
        return new Pair<T>(reference, mark);
    }
}
```

其他的基本一样，这里就不展开了

## 总结

到此为止，基本上Atomic包中常用的工具我们都一一分析完毕，从分析的过程可以看出，Atomic包中的工具大致可以分为以下几类

- AtomicXXX，直接对某个对象进行操作
    - AtomicInteger
    - AtomicBoolean
    - AtomicLong
    - AtomicReference
- AtomicXXArray，对数组进行操作
    - AtomicIntegerArray
    - AtomicLongArray
    - AtomicReferenceArray
- AtomicXXXUpdater，针对对象中的某个字段
    - AtomicIntegerFieldUpdater
    - AtomicLongFieldUpdater
    - AtomicReferenceUpdater
- AtomicXXXReference，提供带版本号的支持
    - AtomicStampedReference
    - AtomicMarkableReference

虽然看起来工具非常多，但是经过分类之后，其用途也就非常明显了，经过分析，我们也能更好地掌握和使用该工具了

