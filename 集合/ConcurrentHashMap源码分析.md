# ConcurrentHashMap实现原理及源码分析

&emsp;&emsp;HashMap是java编程中最常用的数据结构之一，由于HashMap非线程安全，因此不适用于并发访问的场景。JDK1.5之前，通常使用HashTable作为HashMap的线程安全版本，HashTable对读写进行全局加锁，在高并发情况下会造成严重的锁竞争和等待，极大地降低了系统的吞吐量，因此ConcurrentHashMap应运而生。

&emsp;&emsp;相比于Hashtable以及Collections.synchronizedMap()，ConcurrentHashMap在线程安全的基础上提供了更好的写并发能力，并且读操作（ get）通常不会阻塞，使得读写操作可并行执行，支持客户端修改ConcurrentHashMap的并发访问度，迭代期间也不会抛出 `ConcurrentModificationException`等等，但是ConcurrentHashMap有这么多优点，那么它有什么缺点吗？有，一致性问题，这是当前所有分布式系统都面临的问题。ConcurrentHashMap是弱一致性的，具体原因我们后续再分析。

&emsp;&emsp;本文分析的ConcurrentHashMap是基于JDK1.7分析的。

# ConcurrentHashMap实现原理

&emsp;&emsp;ConcurrentHashMap的基本策略是将table细分为多个Segment保存在数组segments中，每个Segment本身又是一个可并发的哈希表，同时每个Segment都是一把ReentrantLock锁，只有在同一个Segment内才存在竞态关系，不同的Segment之间没有锁竞争，这就是分段锁机制。Segment内部拥有一个HashEntry数组，数组中的每个元素又是一个链表。下面看看ConcurrentHashMap整体结构图：

![img-10](https://note.youdao.com/yws/api/personal/file/3D12F6CEF25046B5A4DFBEA480E08942?method=download&shareKey=099548974ad3eb98d34f7969752d677f)

&emsp;&emsp;为了减少占用空间，除了第一个Segment之外，剩余的Segment采用的是延迟初始化的机制，仅在第一次需要时才会创建（通过ensureSegment实现）。为了保证延迟初始化存在的可见性，访问segments数组及table数组中的元素均通过volatile访问，主要借助于Unsafe中原子操作getObjectVolatile来实现，此外，segments中segment的写入，以及table中元素和next域的写入均使用UNSAFE.putOrderedObject来完成。这些操作提供了AtomicReferenceArrays的功能。

> 什么是Unsafe类呢？
>
> - Java 不能直接访问操作系统底层，而是通过本地方法来访问。Unsafe 类提供了硬件级别的原子操作。
> - Unsafe 类在 **sun.misc** 包下，不属于 Java 标准。很多 Java 的基础类库，包括一些被广泛使用的高性能开发库都是基于 Unsafe 类开发，比如 Netty、Hadoop、Kafka 等。
>
> 在ConcurrentHashMap里面很多地方都使用到了Unsafe类，这个类主要是用于直接操作对象等在主内存中的值。
>
> 具体相关可参考：
>
> > ConcurrentHashMap1.7源码分析 https://blog.csdn.net/klordy_123/article/details/82933115
> >
> > 【Java 并发笔记】Unsafe 相关整理 https://www.jianshu.com/p/2e5b92d0962e
> >
> > Unsafe源码 https://my.oschina.net/weichou/blog/704843 
> >
> > C语言内存屏障和volatile关键字：https://www.cnblogs.com/god-of-death/p/7852394.html
> >
> > java内存屏障相关：https://www.jianshu.com/p/c9ac99b87d56 
> >
> >          https://blog.csdn.net/javazejian/article/details/72772470
> >
> >          https://blog.csdn.net/javazejian/article/details/72772461#可见性
> >
> >          https://tech.meituan.com/java_memory_reordering.html

&emsp;&emsp;看到这里我们大概的了解了ConcurrentHashMap的大概构成，也大概知道了ConcurrentHashMap支持高并发的原因，其主要原因是将数据保存到一个 `Segment` 数组中，而每个`Segment`数组均继承自`ReentrantLock`，即每个数组中是一个自带一把锁的数据结构，每当需要修改对应位置内容时，可以先对需要修改的`Segment`加锁，而ConcurrentHashMap默认支持16个 `Segment` ，这就是JDK1.7中ConcurrentHashMap支持高并发的分段锁技术。

&emsp;&emsp;从上面的结构图我们也可以得知，每一个 `Segment` 就是一个哈希数组，也就是说相当于每一个 `Segment` 都是一个小型的HashMap，当然这里的数据结构不是Entry而是**HashEntry**。其解决冲突的思路也是和HashMap类似，使用链地址法来解决hash冲突，即冲突的结构形成链表。下面我们来分析一下ConcurrentHashMap的具体实现源码。

## 成员变量

```java
/**
 * 默认初始容量
 */
static final int DEFAULT_INITIAL_CAPACITY = 16;

/**
 * 默认加载因子
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * 默认并发度，该参数会影响segments数组的长度
 */
static final int DEFAULT_CONCURRENCY_LEVEL = 16;

/**
 * 最大容量，构造ConcurrentHashMap时指定的大小超过该值则会使用该值替换，
 * ConcurrentHashMap的大小必须是2的幂，且小于等于1<<30,以确保可使用int索引条目
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 每个segment中table数组的最小长度，必须是2的幂，至少为2，以免延迟构造后立即调整大小
 */
static final int MIN_SEGMENT_TABLE_CAPACITY = 2;

/**
 * 允许的最大segment数量，用于限定构造函数参数concurrencyLevel的边界，必须是2的幂
 */
static final int MAX_SEGMENTS = 1 << 16; // slightly conservative

/**
 * 非锁定情况下调用size和containsValue方法的重试次数，避免由于table连续被修改导致无限重试
 */
static final int RETRIES_BEFORE_LOCK = 2;
/**
 * 与当前实例相关联的，用于key哈希码的随机值，以减少哈希冲突
 */
private transient final int hashSeed = randomHashSeed(this);

private static int randomHashSeed(ConcurrentHashMap instance) {
    if (sun.misc.VM.isBooted() && Holder.ALTERNATIVE_HASHING) {
        return sun.misc.Hashing.randomHashSeed(instance);
    }
    return 0;
}
/**
 * 用于索引segment的掩码值，key哈希码的高位用于选择segment
 */
final int segmentMask;

/**
 * 用于索引segment偏移值
 */
final int segmentShift;

/**
 * segments数组
 */
final Segment<K,V>[] segments;
```
## 构造方法

```java
/**
 * Creates a new, empty map with the specified initial
 * capacity, load factor and concurrency level.
 *
 * @param initialCapacity the initial capacity. The implementation
 * performs internal sizing to accommodate this many elements.
 * @param loadFactor  the load factor threshold, used to control resizing.
 * Resizing may be performed when the average number of elements per
 * bin exceeds this threshold.
 * @param concurrencyLevel the estimated number of concurrently
 * updating threads. The implementation performs internal sizing
 * to try to accommodate this many threads.
 * @throws IllegalArgumentException if the initial capacity is
 * negative or the load factor or concurrencyLevel are
 * nonpositive.
 */
@SuppressWarnings("unchecked")
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    // 寻找与给定参数concurrencyLevel匹配的最佳Segment数组ssize，必须是2的幂
    // 如果concurrencyLevel是2的幂，那么最后选定的ssize就是concurrencyLevel
    // 否则concurrencyLevel，ssize为大于concurrencyLevel最小2的幂值
    // 即假设concurrencyLevel为7，则ssize为2的3次幂，为8
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //计算每个Segment中，table数组的初始大小
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    // 创建segments和第一个segment
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}

/**
 * Creates a new, empty map with the specified initial capacity
 * and load factor and with the default concurrencyLevel (16).
 *
 * @param initialCapacity The implementation performs internal
 * sizing to accommodate this many elements.
 * @param loadFactor  the load factor threshold, used to control resizing.
 * Resizing may be performed when the average number of elements per
 * bin exceeds this threshold.
 * @throws IllegalArgumentException if the initial capacity of
 * elements is negative or the load factor is nonpositive
 *
 * @since 1.6
 */
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, DEFAULT_CONCURRENCY_LEVEL);
}

/**
 * Creates a new, empty map with the specified initial capacity,
 * and with default load factor (0.75) and concurrencyLevel (16).
 *
 * @param initialCapacity the initial capacity. The implementation
 * performs internal sizing to accommodate this many elements.
 * @throws IllegalArgumentException if the initial capacity of
 * elements is negative.
 */
public ConcurrentHashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}

/**
 * Creates a new, empty map with a default initial capacity (16),
 * load factor (0.75) and concurrencyLevel (16).
 */
/**
 * 默认无参构造器使用的是默认参数
 * DEFAULT_INITIAL_CAPACITY 默认ConcurrentHashMap容量 16
 * DEFAULT_LOAD_FACTOR 负载因子 0.75
 * DEFAULT_CONCURRENCY_LEVEL 默认并发数 16
 */
public ConcurrentHashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}

/**
 * Creates a new map with the same mappings as the given map.
 * The map is created with a capacity of 1.5 times the number
 * of mappings in the given map or 16 (whichever is greater),
 * and a default load factor (0.75) and concurrencyLevel (16).
 *
 * @param m the map
 */
/**
 * map转化为ConcurrentHashMap
 */
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                  DEFAULT_INITIAL_CAPACITY),
         DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
    putAll(m);
}
```
&emsp;&emsp;构造器中各个参数的含义：

**initialCapacity**：创建ConccurentHashMap对象的初始容量，即ConccurentHashMap中HashEntity的总数量，创建时未指定initialCapacity则默认为16，最大容量为MAXIMUM_CAPACITY。

**loadFactor**：负载因子，用于计算Segment的threshold域，

**concurrencyLevel**：即ConccurentHashMap的并发度，支持同时更新ConccurentHashMap且不发生锁竞争的最大线程数。concurrencyLevel不能代表ConccurentHashMap实际并发度，ConccurentHashMap会使用大于等于该值的2的幂指数的最小值作为实际并发度，实际并发度即为segments数组的长度。创建时未指定concurrencyLevel则默认为16。

&emsp;&emsp;并发度对ConccurentHashMap性能具有举足轻重的作用，如果并发度设置的过小，会带来严重的锁竞争问题；如果并发度设置的过大，原本位于同一个Segment内的访问会扩散到不同的Segment中，CPU cache命中率会下降，从而引起程序性能下降。

## 原子方法

话不多说，直接上源码

```java
	/**
     * 获取给定table的第i个元素，使用volatile读语义
     */
    @SuppressWarnings("unchecked")
    static final <K,V> HashEntry<K,V> entryAt(HashEntry<K,V>[] tab, int i) {
        return (tab == null) ? null :
            (HashEntry<K,V>) UNSAFE.getObjectVolatile
            (tab, ((long)i << TSHIFT) + TBASE);
    }

    /**
     * 设置给定table的第i个元素，使用volatile写语义
     */
    static final <K,V> void setEntryAt(HashEntry<K,V>[] tab, int i,
                                       HashEntry<K,V> e) {
        UNSAFE.putOrderedObject(tab, ((long)i << TSHIFT) + TBASE, e);
    }
    /**
     * 通过Unsafe提供的具有volatile元素访问语义的操作获取给定Segment数组的第j个元素（如果ss非空）
     * 注意：因为Segment数组的每个元素只能设置一次（使用完全有序的写入），
     * 所以一些性能敏感的方法只能依靠此方法作为对空读取的重新检查。
     */
    @SuppressWarnings("unchecked")
    static final <K,V> Segment<K,V> segmentAt(Segment<K,V>[] ss, int j) {
        long u = (j << SSHIFT) + SBASE;
        return ss == null ? null :
            (Segment<K,V>) UNSAFE.getObjectVolatile(ss, u);
    }
    /**
     * 根据给定hash获取segment
     */
    @SuppressWarnings("unchecked")
    private Segment<K,V> segmentForHash(int h) {
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        return (Segment<K,V>) UNSAFE.getObjectVolatile(segments, u);
    }

    /**
     * 根据给定segment和hash获取table entry
     */
    @SuppressWarnings("unchecked")
    static final <K,V> HashEntry<K,V> entryForHash(Segment<K,V> seg, int h) {
        HashEntry<K,V>[] tab;
        return (seg == null || (tab = seg.table) == null) ? null :
            (HashEntry<K,V>) UNSAFE.getObjectVolatile
            (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
    }
```

&emsp;&emsp;ConcurrentHashMap主要使用以上几个方法对segments数组和table数组进行读写，并保证线程安全性。

&emsp;&emsp;其主要使用了UNSAFE.getObjectVolatile提供Volatile读语义，UNSAFE.putOrderedObject提供了Volatile写语义。为何要使用UNSAFE来提供volatile语义呢？下面我们来分析一下这两个方法带来的好处：

&emsp;&emsp;UNSAFE.getObjectVolatile使得非volatile声明的对象具有volatile读的语义，那么要使非volatile声明的对象具有volatile写的语义则需要借助操作UNSAFE.putObjectvolatile。

&emsp;&emsp;那么UNSAFE.putOrderedObject操作的含义和作用又是什么呢？

&emsp;&emsp;为了控制特定条件下的指令重排序和内存可见性问题，Java编译器使用一种叫内存屏障（Memory Barrier，或叫做内存栅栏，Memory Fence）的CPU指令来禁止指令重排序。java中volatile写入使用了内存屏障中的LoadStore屏障规则，对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。volatile的写所插入的storeLoad是一个耗时的操作，因此出现了一个对volatile写的升级版本，利用lazySet方法进行性能优化，在实现上对volatile的写只会在之前插入StoreStore屏障，对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见，也就是按顺序的写入。UNSAFE.putOrderedObject正是提供了这样的语义，避免了写指令重排序，但不保证内存可见性，因此读取时需借助volatile读保证可见性。

> 关于内存屏障相关的，可以去查询Java内存模型（JMM）里面关于内存屏障的解释，这里使用内存屏障的原因就是为了保证操作结果不被CPU重排序所影响到。

&emsp;&emsp;ConcurrentHashMap正是利用了这些高性能的原子读写来避免加锁带来的开销，从而大幅度提高了性能。

## ensureSegment方法

ensureSegment方法用于确定指定的Segment是否存在，不存在则会创建。源码如下：

```java
@SuppressWarnings("unchecked")
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

&emsp;&emsp;使用getObjectVolatile()方法提供的原子读语义获取指定Segment，如果为空，以构造ConcurrentHashMap对象时创建的Segment为模板，创建新的Segment。ensureSegment在创建Segment期间为不断使用getObjectVolatile()检查指定Segment是否为空，防止其他线程已经创建了Segment。

## HashEntry数据结构

&emsp;&emsp;HashEntry是ConcurrentHashMap数据结构中最小的存储单元，其内部存储的就是对应key-value值的节点，其源码如下：

```java
/**
 * ConcurrentHashMap list entry. Note that this is never exported
 * out as a user-visible Map.Entry.
 */
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;

    HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    /**
     * Sets next field with volatile write semantics.  (See above
     * about use of putOrderedObject.)
     */
    // 使用volatile写入语义设置next域
    final void setNext(HashEntry<K,V> n) {
        UNSAFE.putOrderedObject(this, nextOffset, n);
    }

    // Unsafe mechanics
    //可以理解为一个指针
    static final sun.misc.Unsafe UNSAFE;
    //偏移量，可以简单的理解为内存地址
    static final long nextOffset;
    static {
        try {
            //获取这个节点对应的内存指针
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class k = HashEntry.class;
            //获取当前节点的next节点对于当前节点指针的偏移量
            nextOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("next"));
            //通过UNSAFE中有方法直接能够获取到当前引用变量的初始内存地址
            //通过初始内存地址和引用变量内部的局部变量的偏移量就可以通过Unsafe直接读取到对应的参数值
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```
&emsp;&emsp;HashEntry将value和next声明为volatile ，是为了保证内存可见性，也就是每次读取都将从内存读取最新的值，而不会从缓存读取。同时，写入next域也使用volatile写入语义保证原子性。写入使用原子性操作，读取使用volatile，从而保证了多线程访问的线程安全性。

## Segment数据结构

&emsp;&emsp;Segment作为ConccurentHashMap的专用数据结构，同时扩展了ReentrantLock，使得Segment本身就是一把可重入锁，方便执行锁定。Segment内部持有一个始终处于一致状态的entry列表，使得读取操作无需加锁（通过volatile读table数组）。调整tables大小期间通过复制节点实现，使得旧版本的table仍然可以遍历。

&emsp;&emsp;Segment仅定义需要加锁的可变方法，针对ConcurrentHashMap中相应方法的调用都会被代理到Segment中的方法。这些可变方法使用scanAndLock和scanAndLockForPut在竞争中使用受控旋转，也就是自旋次数受限制的自旋锁。由于线程的阻塞与唤醒通常伴随着上下文切换、CPU抢占等，都是开销比较大的操作，使用自旋次数受限制的自旋锁，可以提高获取锁的概率，降低线程阻塞的概率，这样可极大地提升性能。之所以自旋次数受限制，是因为自旋会不断的消耗CPU的时间，无限制的自旋会导致开销增长。因此自旋锁适用于多核CPU下，同时线程等待锁的时间非常短；若等待某个锁需要的时间较长，让线程尽早进入阻塞才是正确的选择。下面我们开始来分析源码：

### Segment成员变量

```java
    /**
     * 对segment加锁时，在阻塞之前进行自旋的最大次数。
     * 在多处理器上，使用有限数量的重试来维护在定位节点时获取的高速缓存。
     */
    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

    /**
     * 每个segment的table数组，访问数组中的元素通过entryAt/setEntryAt提供的volatile语义来完成
     */
    transient volatile HashEntry<K,V>[] table;

    /**
     * 元素的数量，只能在锁中或其他volatile读保证可见性之间进行访问
     */
    transient int count;

    /**
     * 当前segment中可变操作发生的次数，put，remove等，可能会溢出32位
     * 它为CHM isEmpty（）和size（）方法中的稳定性检查提供了足够的准确性。
     * 只能在锁中或其他volatile读保证可见性之间进行访问
     */
    transient int modCount;

    /**
     * 当table大小超过此阈值时，对table进行扩容，值为(int)(capacity *loadFactor)
     */
    transient int threshold;

    /**
     * 负载因子
     */
    final float loadFactor;
    /**
     * 构造方法
     */
    Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
        this.loadFactor = lf;
        this.threshold = threshold;
        this.table = tab;
    }
```
### Segment方法

#### scanAndLockForPut方法

我们直接先看源码：

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    //根据key的hash值查找头节点
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    while (!tryLock()) { //尝试获取锁，成功则直接返回，失败则开始自旋
        // 用于后续重新检查头节点
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) { //结束遍历节点
                //e=null代表两种意思，第一种就是遍历链表到了最后，仍然没有发现指定key的entry；
                //第二种情况是刚开始时确实太过entryForHash找到的HashEntry就是空的，即通过hash找到的table中对应位置链表为空
                //当然这里之所以还需要对node==null进行判断，是因为有可能在第一次给node赋值完毕后，然后预热准备工作已经搞定，
                //然后进行循环尝试获取锁，在循环次数还未达到<2>以前，某一次在条件<3>判断时发现有其它线程对这个segment进行了修改，
                //那么retries被重置为-1，从而再一次进入到<1>条件内，此时如果再次遍历到链表最后时，因为上一次遍历时已经给node赋值过了，
                //所以这里判断node是否为空，从而避免第二次创建对象给node重复赋值。
                if (node == null) // 创建节点
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))//找到节点，结束遍历
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {//达到最大尝试次数
            lock();//进入加锁方法，失败则会进入排队，阻塞当前线程
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            // 在遍历过程中，有可能其它线程改变了遍历的链表，这时就需要重新进行遍历了。
            e = first = f; // 头节点变化，需要重新遍历，说明有新节点加入或被移除
            retries = -1;
        }
    }
    return node;
}
```

上面的源码看了之后还是不懂，那么没关系，下面可以看看流程图：

![img-11](https://note.youdao.com/yws/api/personal/file/4C53EB5295E64CA28F3FA57FF95F5856?method=download&shareKey=757f93a7958ae8b8b873798f7c232427)

下面我们分析一下：

&emsp;&emsp;while循环每执行一次，都会尝试获取锁，成功则会返回。retries 初始值设为-1是为了遍历当前hash对应桶的链表，找到则停止遍历，未找到则会预创建一个节点；同时，如果头节点发生变化，则会重新进行遍历，直到自旋次数大于MAX_SCAN_RETRIES，使用lock加锁，获取锁失败则会进入等待队列。

为什么scanAndLockForPut中要遍历一次链表？

&emsp;&emsp;前面已经提过scanAndLockForPut使用自旋次数受限制的自旋锁进行优化加锁的方式，此外，遍历一次链表也是一种优化方法，主要是尽可能使当前链表中的节点进入CPU高速缓存，提高缓存命中率，以便获取锁定后的遍历速度更快。实际上加锁后并没有使用已经找到的节点，因为它们必须在锁定下重新获取，以确保更新的顺序一致性，但是遍历一次后通常可以更快地重新定位。这是一种预热优化的方式，scanAndLock中也使用了该优化方式。

&emsp;&emsp;在Segment中还有一个和这个方法十分类似的scanAndLock方法，其内部实现方式与这个方法相似，但要比这个方法简单，scanAndLock不需要预创建节点。因此scanAndLock主要用于remove和replace操作，而scanAndLockForPut则用于put。这里就不对scanAndLock方法源码进行分析了。

#### Segment的put方法

下面是put的相关源码：

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //先尝试对segment加锁，如果直接加锁成功，那么node=null；如果加锁失败，则会调用scanAndLockForPut方法去获取锁，
    //在这个方法中，获取锁后会返回对应HashEntry（要么原来就有要么新建一个）
    HashEntry<K,V> node = tryLock() ? null :
    scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // 为何这里无需再使用volatile呢？
        // 这里其实是一个优化点，由于table自身是被volatile修饰的，然而put这一块代码本身是加锁了的，所以同一时间内只会有一个线程操作这部分内容
        // 所以不再需要对这一块内的变量做任何volatile修饰，因为变量加了volatile修饰后，变量无法进行编译优化等，会对性能有一定的影响
        // 故将table赋值给put方法中的一个局部变量，从而使得能够减少volatile带来的不必要消耗
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}

final void setNext(HashEntry<K,V> n) {
    UNSAFE.putOrderedObject(this, nextOffset, n);
}
```

上面源码对应的流程图如下：

![img-12](https://note.youdao.com/yws/api/personal/file/4C0ACAFF6FCD488CB001BA6C32972E28?method=download&shareKey=7386fb6d702d7fd7e0d8b57e919ff2a6)

&emsp;&emsp;从上面源码可以看出，其实Segment的put方法和HashMap里面的put方法还是有部分共同之处的，大体流程都是根据 `(table.length-1) & hash` 计算得到相应的索引，然后遍历这个索引对应Entry的链表，进行替换或插入entry操作和rehash操作。

&emsp;&emsp;上面只是大概说了一下HashMap的put和Segment的put方法相似的地方，但是Segment的put方法流程还是稍微多了一些操作。

&emsp;&emsp;Segment中put的大体流程是先对Segment加锁，然后根据`(tab.length-1)&hash`找到对应的slot，然后遍历slot对应的链表，如果key对应的entry已经存在，根据onlyIfAbsent标志决定是否替换旧值，如果key对应的entry不存在，创建新节点插入链表头部，期间若容量超过限制，判断是否需要进行rehash。

&emsp;&emsp;虽然这里的put实现看上去还是相对简单的，但是上面源码中还是有几个优化点值得聊聊：

&emsp;&emsp;scanAndLockForPut的作用上面已经介绍过了，如果锁能很快的获取，有限次数的自旋可防止线程进入阻塞，有助于提升性能；此外，自旋期间会遍历链表，希望遍历的链表被CPU Cache所缓存，为后续实际put过程中的链表遍历操作提升性能；最后scanAndLockForPut还会预创建节点。

&emsp;&emsp;1. HashEntry<K,V>[] tab = table有什么好处？

&emsp;&emsp;上面源码的分析其实已经说得很清楚了，但是在这里还是再啰嗦一下吧，在Segment里面table被声明为volatile，为了保证内存可见性，table上的修改都必须立即更新到主存，volatile写实际是具有一定开销的。由于put中的代码都在加锁区执行，锁既能保证可见性，也能保证原子性，因此，不需要针对table进行volatile写，将table引用赋值给局部变量以实现编译、运行时的优化。

&emsp;&emsp;node.setNext(first)也是同样的道理，HashEntry的next同样被声明为volatile，因此这里使用优化的方式UNSAFE.putOrderedObject进行volatile写入。

&emsp;&emsp;2. 既然put已在加锁区运行，为何访问tab中的元素不直接通过数组索引，而是用entryAt(tab, index)？

&emsp;&emsp;加锁保证了table引用的同步语义，但是对table数组中元素的写入使用UNSAFE.putOrderedObject进行顺序写，该操作只是禁止写写指令重排序，不能保证写入后的内存可见性。因此，必须使用entryAt(tab, index)提供的volatile读来获取最新的数据。

#### remove方法

Segment的remove方法相对比较简单，这里就直接上源码了：

```java
final V remove(Object key, int hash, Object value) {
    if (!tryLock())
        scanAndLock(key, hash);
    V oldValue = null;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> e = entryAt(tab, index);
        HashEntry<K,V> pred = null;
        while (e != null) {
            K k;
            HashEntry<K,V> next = e.next;
            if ((k = e.key) == key ||
                (e.hash == hash && key.equals(k))) {
                V v = e.value;
                if (value == null || value == v || value.equals(v)) {
                    if (pred == null)
                        setEntryAt(tab, index, next);
                    else
                        pred.setNext(next);
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            pred = e;
            e = next;
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

&emsp;&emsp;remove首先会尝试获取锁，失败则会进入scanAndLock代码块，scanAndLock和scanAndLockForPut实现相似。获取锁成功后，然后根据hash找到对应的slot，然后遍历slot对应的链表，找到要移除元素的key，若value为空或与链表中元素的value相等则移除元素，否则退出。

&emsp;&emsp;remove的优化和put中的优化相同，这里就没必要重复说明了。

#### rehash方法

&emsp;&emsp;rehash主要的作用的是扩容，将扩容前table中的节点重新分配到新的table。由于table的capacity都是2的幂，按照2的幂次方扩容为原来的一倍。

&emsp;&emsp;为了提高效率，rehash首先找到第一个后续所有节点在扩容后index都保持不变的节点，将这个节点加入扩容后的table的index对应的slot中，然后将这个节点之前的所有节点重排即可。

```java
/**
 * Doubles size of table and repacks entries, also adding the
 * given node to new table
 * 对数组进行扩容，由于扩容过程需要将老的链表中的节点适用到新数组中，所以为了优化效率，可以对已有链表进行遍历，
 * 对于老的oldTable中的每个HashEntry，从头结点开始遍历，找到第一个后续所有节点在新table中index保持不变的节点fv，
 * 假设这个节点新的index为newIndex，那么直接newTable[newIndex]=fv，即可以直接将这个节点以及它后续的链表中内容全部直接复用copy到newTable中
 * 这样最好的情况是所有oldTable中对应头结点后跟随的节点在newTable中的新的index均和头结点一致，那么就不需要创建新节点，直接复用即可。
 * 最坏情况当然就是所有节点的新的index全部发生了变化，那么就全部需要重新依据k,v创建新对象插入到newTable中。
 */
@SuppressWarnings("unchecked")
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list  单节点
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                // 这个for循环就是找到第一个后续节点新的index不变的节点
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                // 第一个后续节点新index不变节点前所有节点均需要重新创建分配
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    // 添加新节点
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```
关于rehash的流程图如下：

![img-13](https://note.youdao.com/yws/api/personal/file/49ADA77CA9E7434987C93DFEB4838BB8?method=download&shareKey=48244968b55ab37aecd6b9bd98912468)

## ConcurrentHashMap主体代码

&emsp;&emsp;上面分析完HashEntry和Segment的结构和方法，并了解了Unsafe类相关方法后，现在我们就来分析一下ConcurrentHashMap主体的的方法代码，关于ConcurrentHashMap的成员变量因为在前面已经分析过了，所以在这一节里就不多赘述了。不过这里关于ConcurrentHashMap有一点需要知道的是，它内部的Segment数组是延迟创建的，就是说刚开始初始化的时候，只有segments[0]这个Segment是被创建的，其它所有的Segmen均是不会创建的，只有在访问到或者用到这个Segment的时候才会创建，针对这个相关的一个核心方法就是ensureSegment方法。这个方法在上面也已经分析过了，这里也不多做分析。

&emsp;&emsp;上面已经对ConcurrentHashMap主体代码已经铺垫得差不多了，这里主要分析一下ConccurentHashMap中的方法，get、containsKey、containsValue、size、put、putAll、putIfAbsent、remove等等。

### get、containsKey

&emsp;&emsp;get与containsKey两个方法的实现几乎完全一致，都不需要加锁读数据，下面以get源码说明：

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);//获取key对应hash值
    //获取对应h值存储所在segments数组中内存偏移量
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //通过Unsafe中的getObjectVolatile方法进行volatile语义的读，获取到segments在偏移量为u位置的分段Segment
        //并且分段Segment中对应table数组不为空
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
             (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

&emsp;&emsp;首先计算key的hash码，计算Segment的index，使用getObjectVolatile()方法提供的原子读语义获取Segment，再计算Segment中slot的索引，使用getObjectVolatile()方法提供的原子读语义获取slot头节点，遍历链表，判断是否存在key相同的节点以及获得该节点的value。由于遍历过程中其他线程可能对链表结构做了调整，因此get和containsKey返回的可能是过时的数据，这就是ConcurrentHashMap的弱一致性。如果要求强一致性，那么必须使用Collections.synchronizedMap()。

### size、containsValue、isEmpty

&emsp;&emsp;size用于返回ConcurrentHashMap中HashEntry的数量，containsValue用于判断ConcurrentHashMap中是否存在给定的value，isEmpty用于判断ConcurrentHashMap是否为空，size、containsValue、isEmpty都是基于整个ConcurrentHashMap来进行操作的，因此实现原理基本相似。这里以size方法来说明实现原理。size源码如下：

```java
public int size() {
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // 是否溢出，为true表示size溢出32位
    long sum;         // 存储本次循环过程中计算得到的modCount的值
    long last = 0L;   // 存储上一次遍历过程中计算得到的modCount的和
    int retries = -1; // first iteration isn't retry 第一次迭代不计入重试，因此总共会重试3次
    try {
        for (;;) {//无限for循环，结束条件就是任意前后两次遍历过程中modcount值的和是一样的，说明第二次遍历没有做任何变化
            //这里就是前面介绍的为了防止由于有线程不断在更新map而导致每次遍历过程一直发现modCount和上一次不一样
            //从而导致线程一直进行遍历验证前后两次modCount，为了防止这种情况发生，加了一个最多重复的次数限制，
            //超过这个次数则直接强制对所有的segment进行加锁，不过这里需要注意如果出现这种情况，会导致本来要延迟创建的所有segment
            //均在这个过程中被创建
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        //由于只有在retries等于RETRIES_BEFORE_LOCK时才会执行强制加锁，并且由于是用的retries++，
        //所以强制加锁完毕后，retries的值是一定会大于RETRIES_BEFORE_LOCK的，
        //这样就防止正常遍历而没进行加锁时进行锁释放的情况
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

&emsp;&emsp;首先不加锁循环所有的Segment（通过Unsafe的getObjectVolatile()以保证原子读语义），计算所有Segment的count之后，同时计算所有Segment的modcount之和sum，如果sum与last相等，说明迭代期间没有发生其他线程修改ConcurrentHashMap的情况，返回size,；当重试次数超过预定义的值（RETRIES_BEFORE_LOCK为2）时，对所有的Segment依次进行加锁，再计算size的值。需要注意的是，加锁过程中会强制创建所有的不存在Segment，否则容易出现其他线程创建Segment并进行put，remove等操作。由于retries初始值为-1，因此会尝试3次才会对所有的Segment加锁。

&emsp;&emsp;注意：modcount在put, replace, remove以及clear等方法中都会被修改。

&emsp;&emsp;containsValue实现只是在重试问题上稍微不同，在第一次尝试时，sum与last相等也不会返回，默认会尝试试第二次，只有第二次尝试时sum与last也相等才返回。此外，在循环所有的Segment期间，一旦找到匹配value的HashEntry，立即返回，只有value不存在时，才会多次尝试确认。

&emsp;&emsp;isEmpty与containsValue、size实现稍有不同，isEmpty重试失败不会进行加锁，而是直接返回false，isEmpty在循环所有的Segment期间，一旦某个Segment中的entry数量不为0，立即返回true；第一次循环所有的Segment期间，计算每个Segment的modCount之和sum；当sum不为0，进行第二次循环，循环期间，使用sum减去每个Segment的modCount，如果sum不为0，返回false，否则返回true。这样做的好处是，加入第一次循环期间发生了一个put，第二次循环发生了一个remove，那么isEmpty也能检查出当前map为空。

### put、putAll、putIfAbsent

&emsp;&emsp;put和putIfAbsent实现原理基本相似，区别在于当key存在时，put会替换旧值，更新modCount，putIfAbsent则不会。putAll通过循环调用put来实现。这里以put源码来进行说明。

```java
@SuppressWarnings("unchecked")
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          
         (segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

&emsp;&emsp;由源码实现可知，首先计算key的hash码，再计算segments的index，获取Segment，如果Segment为空，则会进入ensureSegment创建Segment，最后将put操作代理给Segment中的put方法实现数据写入。

&emsp;&emsp;注意：put中获取指定Segment没有使用原子读语义，在ensureSegment中会使用原子读语义重新检查。

### remove、replace、clear

&emsp;&emsp;remove和replace都是先通过key计算hash码，定位到Segment，如果Segment为空，则不做任何操作，否则将操作代理给Segment的remove和replace方法。

&emsp;&emsp;clear会循环所有的Segment，如果Segment不空，将操作代理给Segment的clear，Segment的clear操作会直接对Segment加锁，使用UNSAFE.putOrderedObject操作将table数组中的元素置为null。由于clear只是清除元素，Segment指向table的引用不会发生变化，使得clear期间仍然可以进行遍历。

## 弱一致性

&emsp;&emsp;ConcurrentHashMap是弱一致的。 ConcurrentHashMap的get，containsKey，clear，iterator 都是弱一致性的。

&emsp;&emsp;get和containsKey都是无锁的操作，均通过getObjectVolatile()提供的原子读语义来获得Segment以及对应的链表，然后遍历链表。由于遍历期间其他线程可能对链表结构做了调整，因此get和containsKey返回的可能是过时的数据。如在get执行UNSAFE.getObjectVolatile(segments, u)之后，其他线程若执行了clear操作，则get将读到失效的数据。

&emsp;&emsp;由于clear没有使用全局的锁，当清除完一个segment之后，开始清理下一个segment的时候，已经清理segments可能又被加入了数据，因此clear返回的时候，ConcurrentHashMap中是可能存在数据的。因此，clear方法是弱一致的。

&emsp;&emsp;ConcurrentHashMap中的迭代器主要包括entrySet、keySet、values方法，迭代器在遍历期间如果已经遍历的table上的内容变化了，迭代器不会抛出ConcurrentModificationException异常。如果未遍历的数组上的内容发生了变化，则有可能反映到迭代过程中，这就是ConcurrentHashMap迭代器弱一致的表现。

# 参考

> [ConcurrentHashMap实现原理和源码解读](https://my.oschina.net/7001/blog/896587)
>
> [ConcurrentHashMap1.7源码分析](https://blog.csdn.net/klordy_123/article/details/82933115)