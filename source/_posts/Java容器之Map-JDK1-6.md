---
title: Java容器之Map(JDK1.6)
date: 2020-12-11 10:44:44
tags: 
  - Java
  - Collection
  - Map
  - HashMap
  - LinkedHashMap
  - ConcurrentHashMap
  - TreeMap
categories:
  - Java
  - Collections
  - Map
---

# Java容器之Map(JDK1.6)

> Map是Java众多容器家族中一个重要的成员....
>
> Map一个数据结构，对于数据结构而言，最重要的其实就是数据的组织形式，也就是数据在该结构中如何存放，其组织形式也最终决定了其功能、性能，其核心点在于初始化、增加元素、查找元素、删除元素等方面，因此，下面的内容都是围绕这些核心点进行展开

Map接口是Java容器框架重要的一员，其核心是将一个key映射为value，其中key必须是一个可序列化的类型，而value则没有限制，根据实现的不同，Map有几个重要的子类，如

1. 基于Hash算法的HashMap，提供了接近O(1)的查找、修改、删除性能
2. 同样基于Hash算法，并且提供插入顺序或者访问顺序获取的LinkedHashMap
3. 同样基于Hash算法，并且具备并发性能的ConcurrentHashMap
4. 基于红黑树，提供了基于key排序的TreeMap等

当然，此外还有很多基于其他算法的Map，不过最常用到的也就是上面的几种了，这篇文章大致分析了这几个实现的代码包括了结构特点，初始化、插入、查找、删除等

这篇文章基于JDK1.6，原因是1.6的代码比较直观，没有过多的为了性能优化而简化的逻辑，相对来说比较适合入门分析，之后会抽时间再分析1.7、1.8的实现

<!--more-->

## HashMap

### 核心概念

容量：HashMap中桶的数量，默认是16

装填因子：HashMap中元素个数与总大小的比值，默认是0.75f

阈值：当HashMap中元素个数达到某个值时，进行扩容，该值称为阈值(threshold)，数值上=容量*装填因子

### 结构及特点

HashMap是Map的一个实现，其内部结构是一个Entry数组构成的基于链式冲突解决法的hash结构实现，Entry是key-value组成的一个有序对

允许key、value为null，如果key是null，则放置在第一个桶(0号桶)中

### 初始化

HashMap有四种初始化方式，对应四个构造器

1. 无参构造器：使用默认的装填因子、容量、阈值

2. 带容量构造器：使用自定义容量而不是默认容量

3. 带容量、装填因子构造器：使用自定义容量、状态因子初始化

4. 通过其他Map进行初始化

每种初始化方式都会直接或者间接调用一个空方法`init()`，该方法是一个钩子方法，用于子类进行初始化后进行某个特殊的定制操作

其中1、2、3初始化方式类似，只是2、3进行多了参数校验，校验容量是否符合规定(2的X次方)，是否超过最大值，装填因子是否合法等，然后初始化table，也就是桶的大小

方式4比较特殊，是直接从其他的Map进行初始化的，包含了两步

1. 通过map的大小来计算容量，然后调用3使用默认装填因子进行初始化
2. 将map的元素一个个添加到新的Map中：核心代码在`putAllForCreate`这个方法中，如下分析

> putAllForCreate

```Java
private void putAllForCreate(Map<? extends K, ? extends V> m) {
    // 通过Map的entrySet迭代器进行entrySet的迭代
    for (Iterator<? extends Map.Entry<? extends K, ? extends V>> i = m.entrySet().iterator(); i.hasNext(); ) {
        // 对于每一个entry，调用putForCreate将其添加到Map中
        Map.Entry<? extends K, ? extends V> e = i.next();
        putForCreate(e.getKey(), e.getValue());
    }
}
```

> putForCreate

```Java
private void putForCreate(K key, V value) {
    // 如果key是null，则将其hash值设置为0，放置在0号桶中，如果不是，则计算其hash值
    int hash = (key == null) ? 0 : hash(key.hashCode());
    
    // 计算该hash值在桶中的位置
    int i = indexFor(hash, table.length);

    // 从目标桶的第一个元素触发，遍历该链上的每个元素
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        // 如果哈希值相等，并且key不为null并且相等，则将该值设置为对应的值，即替换更新
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {
            e.value = value;
            return;
        }
    }
	
    // 如果在桶中没找着，则将其插入该桶中
    createEntry(hash, key, value, i);
}
```

>createEntry

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
	// 获取头节点
    Entry<K,V> e = table[bucketIndex];
    // 将新增节点作为头结点，并且将原来的头结点设置为当前节点的下一个节点
    table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
    size++;
}
```

> indexFor

```java
static int indexFor(int h, int length) {
    // 这里直接使用位运算，提高处理速度，也是桶数量必须是2的倍数的原因所在
    return h & (length-1);
}
```

### 添加元素

添加元素有两种方式

1. 添加一个元素：put

2. 添加多个元素：putAll

> put

```java
public V put(K key, V value) {
    
    // 如果key是null，则走独立的处理逻辑->放在0号桶
    if (key == null)
        return putForNullKey(value);
    
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    
    // 可以看到，这里跟putForCreate的处理逻辑是类似的
    // 找到目标的桶，然后遍历一下，看看这个桶所在的链上有没有相同的元素，有就替换，没有就添加
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            // 关注下这里，这是LinkedHashMap实现的核心
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    // 这里不是使用前面的createEntry()原因在于createEntry只会在初始化的时候调用，此时已经"扩容"好了
    // 而put的时候，不一定扩容好
    addEntry(hash, key, value, i);
    return null;
}
```

> putForNullKey

```java
private V putForNullKey(V value) {
    // 直接从零号桶遍历列表，找到key=null的
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```

> addEntry

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    // 拿到当前的桶
    Entry<K,V> e = table[bucketIndex];
    // 新建entry并且将其插入到头部(链表的头插法)
    table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
    
    // 如果此时大于阈值，那么就扩容(hashMap中很重要的一个知识点)
    // 扩容是指桶扩容，为原来的2倍
    if (size++ >= threshold)
       resize(2 * table.length);
}
```

> resize

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    
    // 检查是否超过最大容量
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
	
    // 新建桶
    Entry[] newTable = new Entry[newCapacity];
    
    // 将旧桶的数据复制到新桶上
    transfer(newTable);
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}
```

> transfer

```java
void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    // 遍历每个桶
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        
        // 遍历桶上的整个链
        if (e != null) {
            src[j] = null;
            do {
                // 重新计算桶的位置，这里同样采用的是头插法
                // 这里需要特别注意
                // hashMap中著名的多线程导致的CPU空转(CPU狂飙且无法停止)就是在这里发生的
                // 见下面分析
                Entry<K,V> next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

空转问题的发生过程可以参考下面的图，其实，本质上就是因为并发导致的指针指向反转，从而造成了指针收尾相接，造成死循环

![1606466240272](https://raw.githubusercontent.com/xuhuanfeng/images/master/blog-images1606466240272.png)

> putAll

```java
public void putAll(Map<? extends K, ? extends V> m) {
    int numKeysToBeAdded = m.size();
    if (numKeysToBeAdded == 0)
        return;
    // 注意这里不是m.size + size > threshold
    // 原因在于，如果m里头的元素都跟现有map一样，那就会扩容多一倍的空间
    if (numKeysToBeAdded > threshold) {
        // 计算新加入的key所需要的桶数量
        int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
        
        if (targetCapacity > MAXIMUM_CAPACITY)
            targetCapacity = MAXIMUM_CAPACITY;
        
        int newCapacity = table.length;
        // 如果大于现有桶的数量，则进行扩容
        while (newCapacity < targetCapacity)
            newCapacity <<= 1;
        if (newCapacity > table.length)
            resize(newCapacity);
    }

    // 迭代entry，一个个加入map
    for (Iterator<? extends Map.Entry<? extends K, ? extends V>> i = m.entrySet().iterator(); i.hasNext(); ) {
        Map.Entry<? extends K, ? extends V> e = i.next();
        put(e.getKey(), e.getValue());
    }
}
```

### 查找元素

看完了添加元素，其实查找元素就基本上知道是怎么处理了，无非就是先计算key所在的哈希桶，然后遍历该链表，看是否存在，如果存在，返回对应的value，不存在返回null

> get

```java
public V get(Object key) {
    // 如果是key是null，则从零号桶中查找元素
    if (key == null)
        return getForNullKey();
    
    int hash = hash(key.hashCode());
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
            return e.value;
    }
    return null;
}
```

> getForNullKey

```java
private V getForNullKey() {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value;
    }
    return null;
}
```

此外，HashMap还提供一个用于检查是否包含某个值的方法

> containsValue

```java
public boolean containsValue(Object value) {
    if (value == null)
        return containsNullValue();

    Entry[] tab = table;
    // 其实就是遍历每个桶的链表
    for (int i = 0; i < tab.length ; i++)
        for (Entry e = tab[i] ; e != null ; e = e.next)
            if (value.equals(e.value))
                return true;
    return false;
}
```



### 删除元素

删除元素包含两个步骤

1. 查找目标元素
2. 将找到的目标元素从链表中移除

> remove

```java
public V remove(Object key) {
    Entry<K,V> e = removeEntryForKey(key);
    return (e == null ? null : e.value);
}
```

> removeEntryForKey

```java
final Entry<K,V> removeEntryForKey(Object key) {
    int hash = (key == null) ? 0 : hash(key.hashCode());
    int i = indexFor(hash, table.length);
    
    // 通过两个指针的位置变化来实现链表的元素删除
    Entry<K,V> prev = table[i];
    Entry<K,V> e = prev;

    while (e != null) {
        Entry<K,V> next = e.next;
        Object k;
        
        // 如果e所指向的元素就是目标元素
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {
            modCount++;
            size--;
            
            // 如果此时e是第一个元素，那么直接用next替换table[i]就行
            // 如果是中间元素，就通过前一个节点的next=当前节点的next来跳过当前节点
            if (prev == e)
                table[i] = next;
            else
                prev.next = next;
            
            // 留意下这个空方法
            e.recordRemoval(this);
            return e;
        }
        prev = e;
        e = next;
    }

    return e;
}
```

## LinkedHashMap

### 结构及特点

LinkedHashMap是HashMap的一个变种，提供了“有序的”Map能力(本质上是通过额外的指针将所有元素根据指定顺序串联起来，在HashMap中维护多了一个双端链表结构)，HashMap中遍历得到的结果是无序的，而LinkedHashMap则能够提供基于访问/添加的顺序的能力，如按照插入的顺序，或者访问的顺序来获取元素，通过参数：`accessOrder`来控制，可以用LinkedHashMap来实现一个简单的单线程版本的LRU Cache

类签名：

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{

}
```

### 核心概念

由于LinkedHashMap直接继承自HashMap，所以基本同HashMap

LinkedHashMap中增加了一个类型为Entry的`header`元素，用于获取顺序元素的起始点，此外Entry结构比HashMap的多了两个指针：`before`、`after`，用于维护既定顺序

### 初始化

LinkedHashMap提供了五种方式来初始化

1. 指定初始容量、装填因子，同时指定元素顺序为插入顺序
2. 指定初始容量，顺序为插入顺序
3. 采用默认参数，顺序为插入顺序
4. 从其他的Map进行构造，顺序为插入顺序
5. 指定初始容量、装填因子、顺序(true：访问顺序，false：插入顺序)

此外，从前面的HashMap初始化方法中知道，每种方式都会直接或者间接调用`init()`方法来实现子类自己的定制化需求，而LinkedHashMap就用到了这个特性

> init

```java
void init() {
    // 初始化header节点(链表的头结点)
    header = new Entry<K,V>(-1, null, null, null);
    // 初始化header的前后节点信息
    header.before = header.after = header;
}
```

### 添加元素

添加元素本身是直接用父类，也就是HashMap的方法，此外，利用了前面我们提到的`e.recordAccess(this);`来实现按照访问顺序改变节点顺序

> Entry#recordAccess

```java
void recordAccess(HashMap<K,V> m) {
    LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
    if (lm.accessOrder) {
        lm.modCount++;
        // 将当前节点从链表中移除
        remove();
        // 然后将当前节点添加到header前面，成为第一个元素
        addBefore(lm.header);
    }
}
```

同时，前面也看到了createEntry以及addEntry这两个方法，分别用于从其他Map构造的时候使用以及put的时候使用，LinkedHashMap重写了这两个方法，用于实现记录链表顺序

> createEntry

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
       HashMap.Entry<K,V> old = table[bucketIndex];
       Entry<K,V> e = new Entry<K,V>(hash, key, value, old);
       table[bucketIndex] = e;
       // 注意这里，将e添加到header前面(头插法)，于是就成为第一个元素
       e.addBefore(header);
       size++;
}
```

> addEntry

```java
void addEntry(int hash, K key, V value, int bucketIndex) {

    // 先将新元素插入到头节点
    createEntry(hash, key, value, bucketIndex);

    // 然后判断是否需要提出最末尾的元素
    //   header是双端链表的头结点，所以header.after就是链表的末尾节点
    // 如果需要，则移除，不需要则检查是否需要扩容
    Entry<K,V> eldest = header.after;
    
    // 钩子方法，默认是false，LRU的核心
    if (removeEldestEntry(eldest)) {
        removeEntryForKey(eldest.key);
    } else {
        if (size >= threshold)
            resize(2 * table.length);
    }
}
```

> removeEldestEntry

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

在前面我们看到了，如果需要resize的时候，会通过transfer函数来实现扩容的实际操作，这里LinkedHashMap同样覆盖了该方法，提供了自己的实现

> transfer

```java
void transfer(HashMap.Entry[] newTable) {
    int newCapacity = newTable.length;
    
    // 直接遍历双端链表，一个个元素重新插入即可，注意，此时链表本身的顺序没有改变
    // 也就是说，扩容操作不会影响LinkedHashMap的顺序
    for (Entry<K,V> e = header.after; e != header; e = e.after) {
        int index = indexFor(e.hash, newCapacity);
        e.next = newTable[index];
        newTable[index] = e;
    }
}
```

### 查找元素

LinkedHashMap获取元素的方式与HashMap大同小异，只是多了一些额外的操作

> get

```java
public V get(Object key) {
    // 获取节点,父类方法
    Entry<K,V> e = (Entry<K,V>)getEntry(key);
    if (e == null)
        return null;
    // 注意这里，如果是访问顺序，那么这里会移动节点
    e.recordAccess(this);
    return e.value;
}
```

此外LinkedHashMap同样覆盖了containsValue方法，提供的是从header出发遍历的方式来实现检查某个值是否在map中

### 删除元素

LinkedHashMap中移除元素采用的是父类的方法，只不过在移除的时候，需要多移除双端链表的元素，通过`recordRemoval`方法来实现

### LRU Cache的简单实现

LRU全称是Least Recently Used，即最近最少未使用，根据最后一次使用的时间来淘汰数据的缓存过期策略，在看完了LinkedHashMap的实现逻辑之后，我们可以非常方便地使用LinkedHashMap来实现一个单线程版本的简单的LRU

```java
class SimpleLruCache extends LinkedHashMap<String, String> {
    private int maxSize;

    public SimpleLruCache(int initialCapacity, float loadFactor, int maxSize) {
        // accessOrder 设置为true，表示切换为基于访问模式的LinkedHashMap
        super(initialCapacity, loadFactor, true);
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
        // 每次添加时检查一下，如果大于maxSize时，移除最后一个元素
        return size() > maxSize;
    }
}
```

## ConcurrentHashMap

### 结构及特点

前面提到的HashMap和LinkedHashMap本身都不是线程安全的，在多线程环境下会出现各种奇奇怪怪的问题，虽然JDK在之前的版本中提供了HashTable这个工具，不过HashTable本身所有的方法都用`synchronized`进行修饰，这样就意味着无论是读写HashTable的并发度都为1，这在并发度比较高的情况下，性能相对较差

为了应对这个问题，JDK团队提供了ConcurrentHashMap，一个相对来说性能好一些，并且能基本上满足需求的近乎线程安全的hashMap，虽然ConcurrentHashMap也是以HashMap结尾，不过跟前面提到的HashMap本身没有直接的关系，只是他们两者的实现都是基于hash函数来实现散列功能

ConcurrentHashMap实现的原理也是比较好理解的，由于读本身不改变数据元素，不改变结构，所以在读取数据的时候，无需加锁，写操作本身会影响结构，所以需要加锁，但是由于Hash函数本身的特点，只有哈希值一样(哈希冲突)的才会在同一个桶中，所以，理论上来说，访问不同的桶是不相互影响的，也就是说，并发度可以提高到桶数量这个级别

除此之外，HashMap允许key、value为null，而ConcurrentHashMap则不允许

### 核心概念

#### Segment

Segment是ConcurrentHashMap中用于并发管理的基本单元，Segment的数量等于并发度，Segment本身其实也是一个HashMap，只不过具备了并发访问控制能力，所以，在理解的时候，可以直接将其当做HashMap来理解即可，也就是说，其实ConcurrentHashMap是一个两级HashMap，第一级用于控制并发，第二级才是实际存储数据的地方

Segment结构：

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
        transient volatile int count;
        transient int modCount;
    	// 阈值
        transient int threshold;
        transient volatile HashEntry<K,V>[] table;
        final float loadFactor;
}
```

#### HashEntry

HashEntry的结构与HashMap中的Entry基本一致，不过指针字段都用final修饰，确保了强一致性

```java
static final class HashEntry<K,V> {
    final K key;
    final int hash;
    // volatile修饰，保证可见性
    volatile V value;
    final HashEntry<K,V> next;
}
```

#### ConcurrentHashMap

![ConcurrentHashMap结构](https://raw.githubusercontent.com/xuhuanfeng/images/master/blog-images20201203103648.png)

ConcurrentHashMap的结构基本如上图所示，其中需要主义的就是Segment桶是用final修饰的，也就是说，创建完成之后，Segment桶不可变，并发度不可调整，但是Segment里头的哈希桶是可以变更的，也就意味着，哈希扩容是发生在Segment里边的哈希桶，而非Segment桶本身

### 初始化

ConcurrentHashMap一共提供了五种初始化方式

1. 采用默认的初始化容量(默认16)、装填因子(默认0.75)、并发度(默认16)
2. 指定初始容量、其余使用默认
3. 指定初始容量、装填因子、其余默认
4. 指定上述三个参数
5. 从其他Map初始化

前面四种方式最终都是通过4的形式进行初始化，代码如下

```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();

    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;

    // 将并发度调整为2的x次方
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    
    // segmentShift是在计算Segement位置时hash code需要移动的位数
    segmentShift = 32 - sshift;
    // 用于计算所在的Segment桶
    segmentMask = ssize - 1;
    
    this.segments = Segment.newArray(ssize);

    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    
    // 这里是初始化时，每个Segment桶应该初始化的hash桶个数
    // 其实就是总的个数/并发度，然后向上调整为最近的2的X次方
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = 1;
    while (cap < c)
        cap <<= 1;

    // 初始化每一个Segment
    for (int i = 0; i < this.segments.length; ++i)
        this.segments[i] = new Segment<K,V>(cap, loadFactor);
}
```

通过其他Map初始化，有两个步骤

1. 通过4的方式先初始化桶
2. 将map的数据逐个放到桶中(`putAll`方法)

> putAll

```java
public void putAll(Map<? extends K, ? extends V> m) {
    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
        put(e.getKey(), e.getValue());
}
```

> put

```java
// 这里key、value为null都会触发空指针异常
public V put(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key.hashCode());
    // 先计算segment桶的位置，然后委托segment进行put操作
    return segmentFor(hash).put(key, hash, value, false);
}
```

> segmentFor

```java
final Segment<K,V> segmentFor(int hash) {
    // 哈希值先算术右移segmentShift，也就是移动32-并发度
    // 得到的是前(并发度)位的数据，然后&并发度，得到在Segment桶的个数
    return segments[(hash >>> segmentShift) & segmentMask];
}
```

> Segment#put

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 通过ReentrantLock的lock方法进行锁
    // 确保同时只有一个线程在该Segment桶进行操作
    lock();
    try {
        int c = count;
        // 如果大于阈值，则扩容
        if (c++ > threshold) // ensure capacity
            rehash();
        
        HashEntry<K,V>[] tab = table;
        
        // 计算应所在的哈希桶
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        
        // 遍历整条链表，看是否存在
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue;
        // 如果存在
        if (e != null) {
            oldValue = e.value;
            // 并且不是putIfAbsent，则更新该值
            // 如果是，则忽略本次操作
            if (!onlyIfAbsent)
                e.value = value;
        }
        else {
            // 如果不存在，就采用头插法，加入到链表中
            oldValue = null;
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            count = c; // write-volatile
        }
        return oldValue;
    } finally {
        // 释放锁
        unlock();
    }
}
```

> rehash

```java
void rehash() {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity >= MAXIMUM_CAPACITY)
        return;

    // 创建一个新的hash桶，大小为原来的2倍
    HashEntry<K,V>[] newTable = HashEntry.newArray(oldCapacity<<1);
    threshold = (int)(newTable.length * loadFactor);
    int sizeMask = newTable.length - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];

        // 如果原来的桶有元素
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;

            //  并且只有一个元素，则将其当做桶的第一个元素即可
            if (next == null)
                newTable[idx] = e;

            else {
                // 如果有多个元素，由于桶大小是2的X次方，所以很大概率原来在同一个桶的元素
                // resize之后还在同一个桶，如果是这种情况，那么可以减少HashEntry的创建了
                
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                
                // 从第一个元素开始，向后找，找出最后连续N个且在同一个新桶的元素
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                // 这最后N个元素都在同一个桶，直接放到新桶就行了
                newTable[lastIdx] = lastRun;

                // 没有放到新桶的元素，一个个复制一份，然后塞进去
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    int k = p.hash & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(p.key, p.hash,
                                                     n, p.value);
                }
            }
        }
    }
    table = newTable;
}
```

### 添加元素

添加元素有三种方式

1. 直接添加一个元素：put
2. 直接从一个map中添加元素：putAll
3. 如果不存在则放进去，存在则忽略：putIfAbsent

第三种方式是ConcurrentMap接口定义的，目的是为了解决并发环境下Check-And-Set(两个操作步骤)出现的数据不一致问题

> putIfAbsent

```java
public V putIfAbsent(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key.hashCode());
    // 注意这里，最后一个参数onlyIfAbsent是true，表示替换，前面分析了哦
    return segmentFor(hash).put(key, hash, value, true);
}
```

### 查找元素

查找元素有多种方式

1. 根据key获取元素，get
2. 检查是否包含某个key，跟get类似，不分析了
3. 检查是否包含某个value
4. 获取当前大小

> get

```java
public V get(Object key) {
    int hash = hash(key.hashCode());
    return segmentFor(hash).get(key, hash);
}
```

> Segment#get

```java
V get(Object key, int hash) {
    // 注意这里，get方法是没有获取锁的
    if (count != 0) { 
        HashEntry<K,V> e = getFirst(hash);
        while (e != null) {
            // 找到了
            if (e.hash == hash && key.equals(e.key)) {
                V v = e.value;
                // 如果值非空，则直接返回
                if (v != null)
                    return v;
                // 否则，检查一下是否是因为指令重排序引起的(注意这里没有获取锁)
                return readValueUnderLock(e);
            }
            e = e.next;
        }
    }
    return null;
}
```

> Segment#readValueUnderLock

```java
V readValueUnderLock(HashEntry<K,V> e) {
    // 可以看到，其实就是加个锁，重新获取
    lock();
    try {
        return e.value;
    } finally {
        unlock();
    }
}
```

检查是否包含某个值，这里稍微复杂一些

> containsValue

```java
public boolean containsValue(Object value) {
    if (value == null)
        throw new NullPointerException();

    final Segment<K,V>[] segments = this.segments;
    int[] mc = new int[segments.length];

    // 先尝试无锁获取，减少不必要的获取锁操作
    for (int k = 0; k < RETRIES_BEFORE_LOCK; ++k) {
        int sum = 0;
        int mcsum = 0;
        
        // 遍历每个Segment
        for (int i = 0; i < segments.length; ++i) {
            int c = segments[i].count;
            
            // 计算mcsum
            mcsum += mc[i] = segments[i].modCount;
            
            // 如果包含，则必定是包含，直接返回即可
            if (segments[i].containsValue(value))
                return true;
        }
        
        // 如果第一轮没有拿到数据，那么检查下是否是由于并发引起的
        boolean cleanSweep = true;
        if (mcsum != 0) {
            for (int i = 0; i < segments.length; ++i) {
                int c = segments[i].count;
                if (mc[i] != segments[i].modCount) {
                    cleanSweep = false;
                    break;
                }
            }
        }
        // 如果不是并发引起的，说明结果是正确的，就是不存在
        if (cleanSweep)
            return false;
    }
    
    // 如果两轮都没得出正确结果，也就是都是因为并发引起的数据不正确，那么就加锁获取
    for (int i = 0; i < segments.length; ++i)
        segments[i].lock();
    boolean found = false;
    try {
        for (int i = 0; i < segments.length; ++i) {
            if (segments[i].containsValue(value)) {
                found = true;
                break;
            }
        }
    } finally {
        for (int i = 0; i < segments.length; ++i)
            segments[i].unlock();
    }
    return found;
}
```

或者size等操作与container基本类似，就不展开了

### 删除元素

删除元素有两种操作

1. 根据key删除
2. 根据key、value删除，只有都匹配才删除

> remove

```java
public V remove(Object key) {
int hash = hash(key.hashCode());
       return segmentFor(hash).remove(key, hash, null);
   }
```

> Segment#remove

```java
V remove(Object key, int hash, Object value) {
    lock();
    try {
        int c = count - 1;
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue = null;
        if (e != null) {
            V v = e.value;
            if (value == null || value.equals(v)) {
                oldValue = v;
                ++modCount;
                HashEntry<K,V> newFirst = e.next;
                
                // 由于HashEntry中的next字段是final类型，所以目标节点前面的元素
                // 没有办法跟hashMap一样，直接通过next = next.next来断开节点
                // 只能是重新复制一份，然后重新链接回链表中
                
                // 没有人链接的节点，即e，由于gc root不可达，所以会被gc
                for (HashEntry<K,V> p = first; p != e; p = p.next)
                    newFirst = new HashEntry<K,V>(p.key, p.hash,
                                                  newFirst, p.value);
                tab[index] = newFirst;
                count = c;
            }
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

## TreeMap

### 结构及特点

JDK中的TreeMap是红黑树的实现，关于红黑树，可以先看下这篇[红黑树学习笔记]([红黑树学习笔记 | 凌风的小窝 (xuhuanfeng.cn)](http://www.xuhuanfeng.cn/2020/12/06/%E7%BA%A2%E9%BB%91%E6%A0%91%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/))，强烈建议读一下《Algorithm 4th》中相关的章节

TreeMap会对Key进行排序，也意味着，TreeMap是一个有序的Map，同时，为了维护顺序，必然也会带来一些性能的损耗(相比于HashMap)

TreeMap实现了NavigableMap，提供了获取大于/小于某个key的Entry，获取降序map等等跟排序有关的能力，进一步扩充了Map本身的能力

![TreeMap继承关系](https://raw.githubusercontent.com/xuhuanfeng/images/master/blog-images20201208102123.png)

### 核心操作

TreeMap是一个红黑树，红黑树是BST的一个变种，所以TreeMap本身就是一个BST，那么对于TreeMap的各种操作，实际上就是对应BST的各种操作

在BST中，中序遍历得到的结果就是一个已经排好序的结果，而在TreeMap中，我们无论是通过迭代器，还是其他的方式获取Map的元素列表，最终其实都是在访问BST，并且是以中序的方式访问

在BST的访问中，有几个比较重要的操作，如获取第一个元素，获取最后一个元素，获取某个元素的前一个元素，获取某个元素的下一个元素等，这些特性本身也是有序带来的特点

#### 第一个元素

在二叉树中，第一个元素是根节点，也就是root节点，然而，在BST中则不是，BST中的第一个元素是指值最小的那个元素，这个元素会出现在一下几个地方

1. 如果root为null，则不存在(树本身都不存在)
2. 如果左节点不为空，则一直往最节点走，目标就是最左节点的最左节点
3. 如果最左节点不存在，则root本身就是目标元素

```java
final Entry<K,V> getFirstEntry() {
    Entry<K,V> p = root;
    // root为空
    if (p != null)
        // 如果左节点不为空，则一直往左走，否则，当前节点就是目标(第一个元素)
        while (p.left != null)
            p = p.left;
    return p;
}
```

#### 最后一个元素

最后一个元素与第一个元素相反，位于最右子树的最右节点

```java
final Entry<K,V> getLastEntry() {
    Entry<K,V> p = root;
    if (p != null)
        // 一直往右走
        while (p.right != null)
            p = p.right;
    return p;
}
```

#### 某个元素的下一个元素

某个元素的下一个元素同样的，是指在排序中位于该元素的下一个元素，他们可能并不是直接连接在一起的，也即，root的right并不一定就是root的下一个元素

下一个元素的查找算法：

1. 如果右节点存在，目标节点为：右节点的最左节点
2. 如果右节点不存在，则为沿着根路径往上走，直到出现“右拐”节点(第一个比本节点大的节点)，该节点就是目标节点

```java
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.right != null) {
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else {
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        // 往上走，找到第一个右拐点
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```

#### 某个元素的前一个节点

前一个节点查找算法

1. 如果左节点不为空，则目标节点为左节点的最右节点
2. 如果左节点为空，则沿着根节点往上走，直到第一个“左拐”节点出现

```java
static <K,V> Entry<K,V> predecessor(Entry<K,V> t) {
    if (t == null)
        return null;
    // 左节点的最右节点
    else if (t.left != null) {
        Entry<K,V> p = t.left;
        while (p.right != null)
            p = p.right;
        return p;
    } else {
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        // 沿着根节点出发，找到第一个”左拐“节点
        while (p != null && ch == p.left) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```

### 初始化

TreeMap提了四种初始化方式

1. 使用key的默认排序方式
2. 自定义key排序器
3. 从map接口的实例初始化，同时使用key的默认排序方式
4. 从SortedMap初始化(注意与3的不同)

从map接口的实例初始化，会先检查下是否是SortedMap的实例，如果是的话，会通过4进行初始化，如果不是的话，就直接通过父类(`AbstractMap`)的putAll操作，最终会回到自己的put方法

> putAll

```java
public void putAll(Map<? extends K, ? extends V> map) {
      int mapSize = map.size();
      // size == 0表示初始化
      // 新的map非空，并且是SortedMap的实例
      if (size==0 && mapSize!=0 && map instanceof SortedMap) {
          // 获取map的排序器
          Comparator c = ((SortedMap)map).comparator();
          // 如果跟自己的排序器一样，那说明符合自己的排序规则
          if (c == comparator 
                || (c != null && c.equals(comparator))) {
              ++modCount;
          try {
                 buildFromSorted(mapSize, map.entrySet().iterator(),
               null, null);
               } catch (java.io.IOException cannotHappen) {
               } catch (ClassNotFoundException cannotHappen) {
          }
          return;
      }
  }
    // 如果不是初始化，或者排序器不一样，则直接通过父类的putAll进行操作
    // 父类的putAll会遍历map，然后一个个调用put操作
  super.putAll(map);
}
```

> buildFromSorted

```java
private void buildFromSorted(int size, Iterator it,
 java.io.ObjectInputStream str, V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    this.size = size;
    // 构建红黑树
    root = buildFromSorted(0, 0, size-1, computeRedLevel(size),
      it, str, defaultVal);
}
```

> computeRedLevel

```java
// level = log(sz)
private static int computeRedLevel(int sz) {
    int level = 0;
    for (int m = sz - 1; m >= 0; m = m / 2 - 1)
        level++;
    return level;
}
```

> buildFromSorted

```java

// itr 存储是树的中序遍历(BST的中序遍历得到的序列就是有序)
// 由于中序遍历的时候，根节点在最中间，左子树在根的左边，右子树在根的右边
// 因此，重新构建树的时候，需要先构建左子树，构建根，然后构建右子树
private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
        int redLevel,
        Iterator it,
        java.io.ObjectInputStream str,
        V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    
    if (hi < lo) return null;

    int mid = (lo + hi) / 2;

    Entry<K,V> left  = null;
    
    // 递归构建左子树
    if (lo < mid)
        left = buildFromSorted(level+1, lo, mid - 1, redLevel,
   it, str, defaultVal);

    K key;
    V value;
    //如果it非空，则从it取数据，否则从stream取数据
    if (it != null) {
        if (defaultVal==null) {
            Map.Entry<K,V> entry = (Map.Entry<K,V>)it.next();
            key = entry.getKey();
            value = entry.getValue();
        } else {
            key = (K)it.next();
            value = defaultVal;
        }
    } else { 
        key = (K) str.readObject();
        value = (defaultVal != null ? defaultVal : (V) str.readObject());
    }

    // 到这里左子树已经构建完成，构建根节点
    Entry<K,V> middle =  new Entry<K,V>(key, value, null);

    // 如果当前节点的层次为最低层，则将节点设置为红色
    if (level == redLevel)
        middle.color = RED;

    // 构建左节点
    if (left != null) {
        middle.left = left;
        left.parent = middle;
    }

    // 构建右节点
    if (mid < hi) {
        Entry<K,V> right = buildFromSorted(level+1, mid+1, hi, redLevel,
          it, str, defaultVal);
        middle.right = right;
        right.parent = middle;
    }

    // 返回当前子树
    return middle;
}
```

到这里，从其他的排序树构建的过程就结束了，接下来看下一个个元素构建的过程，也就是put方法

> put

```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    // 如果root为null，则构建root即可
    if (t == null) {
        root = new Entry<K,V>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    
    Comparator<? super K> cpr = comparator;
    
    // 如果比较器非空，说明自定义了key的比较方式
    if (cpr != null) {
        do {
            // 记录下parent
            parent = t;
            // 如果小，则往左走
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            // 如果大，则往右走
            else if (cmp > 0)
                t = t.right;
            // 否则，直接替换
            else
                return t.setValue(value);
        } while (t != null);
    }
    // 采用key默认的比较方式
    else {
        if (key == null)
            throw new NullPointerException();
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    
    // 初始化当前节点
    Entry<K,V> e = new Entry<K,V>(key, value, parent);
    
    // 作为左节点
    if (cmp < 0)
        parent.left = e;
    // 作为右节点
    else
        parent.right = e;
    
    // 由于加入新节点之后，可能会出现违背红黑树规则的情况
    // 因此，需要调整(旋转、变色)下树的结构，以保障红黑树的特性
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

> fixAfterInsertion

```java
private void fixAfterInsertion(Entry<K,V> x) {
    // 新加入的节点默认是红节点
    x.color = RED;
	
    // 如果父节点也是红色，则违背了红黑树“红色节点的子节点均为黑色节点”的原则
    // 通过 1：旋转，2：变色进行处理
    while (x != null && x != root && x.parent.color == RED) {
        // 如果x的父节点(记为p)是p的父节点的左节点，也就是红色的父节点位于祖父节点的左侧
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // 获取p的兄弟节点
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            // case 1:
            // 如果p的兄弟节点y也是红色
            if (colorOf(y) == RED) {
                // 直接将p以及y变为黑色，然后父节点变为红色
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                // 沿着根路径继续向上
                x = parentOf(parentOf(x));
            } else {
                
                // case 2:
                // y是黑色
                
                // x在p的右侧
                if (x == rightOf(parentOf(x))) {
                    // 父节点先左旋，于是就变成了两个红色节点都位于左侧
                    x = parentOf(x);
                    // 此时x指向原来的p
                    rotateLeft(x);
                }
                // x节点位于p的左侧
                
                // x父节点变成黑色
                setColor(parentOf(x), BLACK);
                
                // x的父节点的父节点变成红色
                setColor(parentOf(parentOf(x)), RED);
                // x的父节点的父节点右旋
                rotateRight(parentOf(parentOf(x)));
                // 此时就转换为了 case1，在循环中会继续进行变色
            }
        } else {
            
            // x的父节点p，是p的父节点的右节点
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            
            // case 3：
            // 如果p的兄弟节点，是红色，其实就回到了case 1，直接变色即可
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                // case 4:
                // 如果x是p的左节点，其实是case2的镜像，就进行右旋
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                // 此时是p以及x都是右节点，且都是红节点
                
                // 将p设置为黑色
                setColor(parentOf(x), BLACK);
                
                // 将p的父节点设置为红色
                setColor(parentOf(parentOf(x)), RED);
                
                // 将p的父节点进行右旋
                rotateLeft(parentOf(parentOf(x)));
                
                // 此时转换为了case 1
            }
        }
    }
    // 将根的颜色重置为黑色
    root.color = BLACK;
}
```

到这里，TreeMap的初始化也就完成了，其实，添加元素也完成了

### 添加元素

见上面的put、putAll分析

### 查找元素

查找元素有多种方式，比如查找某个key，查找第一个大于/小于该key的节点等等

> get

```java
public V get(Object key) {
    Entry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}
```

> getEntry

```java
final Entry<K,V> getEntry(Object key) {
       // 如果comparator非空，则使用自定义的comparator来查找元素
       if (comparator != null)
           return getEntryUsingComparator(key);
    
       // 如果key为null，非法
       if (key == null)
           throw new NullPointerException();
Comparable<? super K> k = (Comparable<? super K>) key;
       Entry<K,V> p = root;
    
       // 使用key自定义的排序器进行查找，简单的BST查找，就不展开了
       while (p != null) {
           int cmp = k.compareTo(p.key);
           if (cmp < 0)
               p = p.left;
           else if (cmp > 0)
               p = p.right;
           else
               return p;
       }
       return null;
   }
```

> getEntryUsingComparator

```java
// BST查找，只是使用自定义的排序器
final Entry<K,V> getEntryUsingComparator(Object key) {
       K k = (K) key;
       Comparator<? super K> cpr = comparator;
       if (cpr != null) {
           Entry<K,V> p = root;
           while (p != null) {
               int cmp = cpr.compare(k, p.key);
               if (cmp < 0)
                   p = p.left;
               else if (cmp > 0)
                   p = p.right;
               else
                   return p;
           }
       }
       return null;
   }
```

其他的方法就根据使用的时候，如果不理解再进行分析即可，就不展开了

### 删除元素

在HashMap中，删除元素只需要通过改变指针的指向就能解决，而在LinkedHashMap中，无非也就多了额外的指针变化，然而，在TreeMap中，情况则复杂很多，由于删除之后，可能导致红黑树不平衡了，所以，还需要进一步进行平衡操作

> remove

```java
public V remove(Object key) {
    
    // 找不到就说明不存在，无需删除
    Entry<K,V> p = getEntry(key);
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p);
    return oldValue;
}
```

> deleteEntry

```java
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // 如果左右节点都非空
    if (p.left != null && p.right != null) {
        // 查找p的下一个元素(还记得这个方法的实现吗？)
        Entry<K,V> s = successor (p);
        // 将该节点值方在p中，并且将p指正s，此时变成了移除s
        p.key = s.key;
        p.value = s.value;
        p = s;
    } 

    // 查找原来的s，现在的p的后继(左儿子，没有则找右儿子)，用于替换p
    // 注意这里，此时p只有左或者右节点，不存在两者都有的情况
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);

    // 如果存在，也即p不是叶子节点
    if (replacement != null) {
        
        replacement.parent = p.parent;
        // 如果p没有父节点，说明p是root，那么此时replacement将成为新的root
        if (p.parent == null)
            root = replacement;
        // 如果p是左节点，则将replacement作为左节点
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        // 作为右节点
        else
            p.parent.right = replacement;

        // 清空p的连接信息
        p.left = p.right = p.parent = null;

        // 如果p(替换节点)是黑色节点，那么可能会违背“从root到叶子节点黑色节点路径长度一样”的规则
        // 所以需要进行修正，注意，是用替换节点来修正(可能违背红黑规则)
        // 此时replacement已经替换了p，直接从replacement开始修正
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
        
     // 如果没有后继节点，且没有父节点，说明自己是孤家寡人
    } else if (p.parent == null) { 
        root = null;
    } else { 
        // 没有后继节点，但是自己是黑色，也同样需要进行修正
        if (p.color == BLACK)
            fixAfterDeletion(p);

        // 如果存在父节点，清空父节点的左/右节点信息
        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```

> fixAfterDeletion

```java
// 修正节点信息
// todo 好吧，这里实在是没看懂.....后面有机会再补全吧...
private void fixAfterDeletion(Entry<K,V> x) {
    // 非root节点并且是黑色节点
    while (x != root && colorOf(x) == BLACK) {
        // 如果是左节点
        if (x == leftOf(parentOf(x))) {
            
            Entry<K,V> sib = rightOf(parentOf(x));

            // 兄弟节点是红色的
            // 违背了规则，变色，旋转
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }

            // 如果调整后的兄弟节点左右都为黑色，则将其变色为红色
            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric
            Entry<K,V> sib = leftOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```

