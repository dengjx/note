# HashMap实现原理及源码分析
&emsp;&emsp;哈希表（hash table）也叫散列表，是一种非常重要的数据结构，应用场景及其丰富，许多缓存技术（比如memcached）的核心其实就是在内存中维护一张大的哈希表。这里的hashMap基于JDK1.7进行分析。

注意！JDK1.7和JDK1.8在HashMap的实现上面差别较大，JDK1.8对HashMap进行了优化，详情可以查看《==JDK1.7和JDK1.8 HashMap比较==》这篇文章。

- **目录**
    + [一、什么是哈希表](#1)
    + [二、HashMap实现原理](#2)
    + [三、为何HashMap的数组长度一定是2的次幂？](#3)
    + [四、重写equals方法需同时重写hashCode方法](#4)

## <span id='1'>一、什么是哈希表</span>
&emsp;&emsp;在讨论哈希表之前，我们先大概了解下其他数据结构在新增，查找等基础操作执行性能

&emsp;&emsp;**数组：** 采用一段连续的存储单元来存储数据。对于指定下标的查找，时间复杂度为O(1)；通过给定值进行查找，需要遍历数组，逐一比对给定关键字和数组元素，时间复杂度为O(n)，当然，对于有序数组，则可采用二分查找，插值查找，斐波那契查找等方式，可将查找复杂度提高为O(logn)；对于一般的插入删除操作，涉及到数组元素的移动，其平均复杂度也为O(n)

&emsp;&emsp;**线性链表：** 对于链表的新增，删除等操作（在找到指定操作位置后），仅需处理结点间的引用即可，时间复杂度为O(1)，而查找操作需要遍历链表逐一进行比对，复杂度为O(n)

&emsp;&emsp;**二叉树：** 对一棵相对平衡的有序二叉树，对其进行插入，查找，删除等操作，平均复杂度均为O(logn)。

&emsp;&emsp;**哈希表：** 相比上述几种数据结构，在哈希表中进行添加，删除，查找等操作，性能十分之高，不考虑哈希冲突的情况下，仅需一次定位即可完成，时间复杂度为O(1)，接下来我们就来看看哈希表是如何实现达到惊艳的常数阶O(1)的。

&emsp;&emsp;我们知道，数据结构的物理存储结构只有两种：**顺序存储结构**和**链式存储结构**（像栈，队列，树，图等是从逻辑结构去抽象的，映射到内存中，也这两种物理组织形式），而在上面我们提到过，在数组中根据下标查找某个元素，一次定位就可以达到，哈希表利用了这种特性，**哈希表的主干就是数组**。

比如我们要新增或查找某个元素，我们通过把当前元素的关键字 通过某个函数映射到数组中的某个位置，通过数组下标一次定位就可完成操作。

```
存储位置 = f(关键字)
```

其中，这个函数f一般称为哈希函数，这个函数的设计好坏会直接影响到哈希表的优劣。举个例子，比如我们要在哈希表中执行插入操作：
![img-5](https://note.youdao.com/yws/api/personal/file/A2665970ED0F4FC78D564CC81F402E7D?method=download&shareKey=5f9f55c63b464d3b682a6ae32a9e2457)
查找操作同理，先通过哈希函数计算出实际存储地址，然后从数组中对应地址取出即可。

**哈希冲突**

&emsp;&emsp;然而万事无完美，如果两个不同的元素，通过哈希函数得出的实际存储地址相同怎么办？也就是说，当我们对某个元素进行哈希运算，得到一个存储地址，然后要进行插入的时候，发现已经被其他元素占用了，其实这就是所谓的哈希冲突，也叫哈希碰撞。前面我们提到过，哈希函数的设计至关重要，好的哈希函数会尽可能地保证 计算简单和散列地址分布均匀,但是，我们需要清楚的是，数组是一块连续的固定长度的内存空间，再好的哈希函数也不能保证得到的存储地址绝对不发生冲突。那么哈希冲突如何解决呢？哈希冲突的解决方案有多种:开放定址法（发生冲突，继续寻找下一块未被占用的存储地址），再散列函数法，链地址法，而HashMap即是采用了链地址法，也就是**数组+链表**的方式，

## <span id='2'>二、HashMap实现原理</span>
HashMap的主干是一个Entry数组。Entry是HashMap的基本组成单元，每一个Entry包含一个key-value键值对。

```java
/**
 * The table, resized as necessary. Length MUST Always be a power of two.
 */
//HashMap的主干数组，可以看到就是一个Entry数组，初始值为空数组{}，主干数组的长度一定是2的次幂
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
```

Entry是HashMap中的一个静态内部类。相关声明如下：

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;

    /**
     * Creates new entry.
     */
    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }
 
    //其他代码
    ......
}
```

所以，HashMap的整体结构如下
![img-6](https://note.youdao.com/yws/api/personal/file/F2FB1D071A234B1D83C3908D447ADBB9?method=download&shareKey=b1ac4a9208f49a490bc0e0783ddae613)

**简单来说，HashMap由数组+链表组成的，数组是HashMap的主体，链表则是主要为了解决哈希冲突而存在的，如果定位到的数组位置不含链表（当前entry的next指向null）,那么对于查找，添加等操作很快，仅需一次寻址即可；如果定位到的数组包含链表，对于添加操作，其时间复杂度为O(n)，首先遍历链表，存在即覆盖，否则新增；对于查找操作来讲，仍需遍历链表，然后通过key对象的equals方法逐一比对查找。所以，性能考虑，HashMap中的链表出现越少，性能才会越好。**

其他几个重要字段

```java
/**
 * The default initial capacity - MUST be a power of two.
 */
//默认初始化数组容量，默认是16，而且容量必须是2次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
//最大数组容量
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The load factor used when none specified in constructor.
 */
//默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * The number of key-value mappings contained in this map.
 */
//实际存储的key-value键值对的个数
transient int size;

/**
 * The next size value at which to resize (capacity * load factor).
 * @serial
 */
// If table == EMPTY_TABLE then this is the initial capacity at which the
// table will be created when inflated.
//阈值，当table == {}时，该值为初始容量（初始容量默认为16）
//当table被填充了，也就是为table分配内存空间后，threshold一般为 capacity * loadFactory。
//HashMap在进行扩容时需要参考threshold
int threshold;

/**
 * The load factor for the hash table.
 *
 * @serial
 */
//负载因子，代表了table的填充度有多少，默认是0.75
final float loadFactor;

/**
 * The number of times this HashMap has been structurally modified
 * Structural modifications are those that change the number of mappings in
 * the HashMap or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the HashMap fail-fast.  (See ConcurrentModificationException).
 */
/**
 * 用于快速失败，由于HashMap非线程安全，在对HashMap进行迭代时，
 * 如果期间其他线程的参与导致HashMap的结构发生变化了（比如 put，remove等操作），
 * 需要抛出异常ConcurrentModificationException 
 */
transient int modCount;
```

HashMap有4个构造器，其他构造器如果用户没有传入initialCapacity 和loadFactor这两个参数，会使用默认值

initialCapacity默认为16，loadFactory默认为0.75

```java
/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
    //此处对传入的初始容量进行校验，最大不能超过MAXIMUM_CAPACITY = 1<<30(2^30)
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);

    this.loadFactor = loadFactor;
    threshold = initialCapacity;
    init();
}
```

从上面这段代码我们可以看出，在HashMap的常规构造器中，没有为数组table分配内存空间（除入参为Map的构造器以外），而是在执行put操作的时候才真正构建table数组。

### 2.1 put方法
下面我们来分析一下put方法：

```java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
 *         (A <tt>null</tt> return can also indicate that the map
 *         previously associated <tt>null</tt> with <tt>key</tt>.)
 */
public V put(K key, V value) {
    /**
     * 如果table数组为空数组{}，进行数组填充（为table分配实际内存空间），
     * 入参为threshold，此时threshold为initialCapacity 默认是1<<4(24=16)
     */
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    //如果key为null，存储位置为table[0]或table[0]的冲突链上
    if (key == null)
        return putForNullKey(value);
    //计算出hash值，保证散列均匀
    int hash = hash(key);
    //获取在数组中的实际存储索引值
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        //如果该对应数据已存在，执行覆盖操作。用新 value替换旧 value，并返回旧 value
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    //保证并发访问时，若HashMap内部结构发生变化，快速响应失败
    modCount++;
    //插入一个新的entry
    addEntry(hash, key, value, i);
    return null;
}
```

这里的操作大概看不懂的话别着急，下面我们来一步步分析下去，我们先来看看inflateTable这个方法：

```java
/**
 * Inflates the table.
 */
//扩展table方法
private void inflateTable(int toSize) {
    // Find a power of 2 >= toSize
    // capacity容量一定是2次幂
    int capacity = roundUpToPowerOf2(toSize);
    /**
     * 此处为threshold赋值，取capacity*loadFactor和MAXIMUM_CAPACITY+1的最小值，
     * capaticy一定不会超过MAXIMUM_CAPACITY，除非loadFactor大于1
     */
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    initHashSeedAsNeeded(capacity);
}
```

inflateTable这个方法用于为主干数组table在内存中分配存储空间，通过roundUpToPowerOf2(toSize)可以确保capacity为大于或等于toSize的最接近toSize的二次幂，比如toSize=13，则capacity=16；to_size=16，capacity=16；to_size=17，capacity=32。

roundUpToPowerOf2方法如下：

```java
/**
 * roundUpToPowerOf2中的这段处理使得数组长度一定为2的次幂，Integer.highestOneBit
 * 是用来获取最左边的 bit（其他 bit位为0）所代表的数值
 */
private static int roundUpToPowerOf2(int number) {
    // assert number >= 0 : "number must be non-negative";
    return number >= MAXIMUM_CAPACITY
            ? MAXIMUM_CAPACITY
            : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
}
```

roundUpToPowerOf2中的这段处理使得数组长度一定为2的次幂，Integer.highestOneBit是用来获取最左边的bit（其他bit位为0）所代表的数值。

下面，我们来看看hash值是如何计算出来的，尽量保证不碰撞以及散列均匀

```java
/**
 * A randomizing value associated with this instance that is applied to
 * hash code of keys to make hash collisions harder to find. If 0 then
 * alternative hashing is disabled.
 */
// 这是一个随机值，用于使得 hash冲突更难出现，0是禁用可选 hash
transient int hashSeed = 0;

/**
 * Retrieve object hash code and applies a supplemental hash function to the
 * result hash, which defends against poor quality hash functions.  This is
 * critical because HashMap uses power-of-two length hash tables, that
 * otherwise encounter collisions for hashCodes that do not differ
 * in lower bits. Note: Null keys always map to hash 0, thus index 0.
 */
/**
 * 这个方法里用了大量的异或和移位运算，对 key的 hashCode进一步处理，
 * 保证最终计算出来的存储位置尽可能分散。
 */
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

通过这个hash方法计算出来的hash值并不是最终的存储位置，还需要经过最后一步indexFor方法才能最终得到存储在数组中的位置。相关源码如下：

```java
/**
 * Returns index for hash code h.
 */
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    // 可以断言，length的二进制表示中的1数量必定等于1，也就是长度必须为非零的 2 次幂
    // 用计算出来的 hash和数组的 length - 1 进行与运算
    return h & (length-1);
}
```

我们想一下如何保证计算出来的索引值在数组的长度范围内呢？

`h & (length-1)` 保证获取的index一定在数组范围内，我们可以举个例子，假设使用默认容量16，length-1=15，h=20，换算成二进制进行计算可得到：

```
        0 1 1 1 1        15
     &  1 0 1 0 0        20
     -------------
        0 0 1 0 0      = 4
```

最终计算出来的index=4，有些版本在这里会进行 取模运算，这样也能保证index一定在数组范围内，不过位运算对于计算机来说，性能更高一些。(HashMap中有大量的位运算)。

分析到这里，上面的问题应该就有所想法了吧，这里也能知道为何数组的长度必须为2次幂了，因为如果长度是2次幂的话，换成二进制的时候必定只有一位是1，其余为0，当进行减1操作的时候，会得到除了一个0以外其余都为1的二进制数，这样可以保证进行与运算后得到的数必定小于这个length长度。

所以存储位置的确定流程是这样的：

```
graph LR
A[Key]-->|hashCode方法| B[hashCode]
B[hashCode]-->|hash方法| C[h]
C[h]-->|h&length-1| D[存储下标]
```

接着，我们来看看addEntry方法的实现：

```java
/**
 * Adds a new entry with the specified key, value and hash code to
 * the specified bucket.  It is the responsibility of this
 * method to resize the table if appropriate.
 *
 * Subclass overrides this to alter the behavior of put method.
 */
void addEntry(int hash, K key, V value, int bucketIndex) {
    ////当size超过临界阈值threshold，并且即将发生哈希冲突时进行扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length); 
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length); //重新计算即将冲突的索引值
    }

    createEntry(hash, key, value, bucketIndex);
}
```

通过以上代码能够得知，当发生哈希冲突并且size大于阈值的时候，需要进行数组扩容，扩容时，需要新建一个长度为之前数组2倍的新的数组，然后将当前的Entry数组中的元素全部传输过去，扩容后的新数组长度为之前的2倍，所以扩容相对来说是个耗资源的操作。

### 2.2 get方法
下面我们来分析一下get方法：

```java
/**
 * Returns the value to which the specified key is mapped,
 * or {@code null} if this map contains no mapping for the key.
 *
 * <p>More formally, if this map contains a mapping from a key
 * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
 * key.equals(k))}, then this method returns {@code v}; otherwise
 * it returns {@code null}.  (There can be at most one such mapping.)
 *
 * <p>A return value of {@code null} does not <i>necessarily</i>
 * indicate that the map contains no mapping for the key; it's also
 * possible that the map explicitly maps the key to {@code null}.
 * The {@link #containsKey containsKey} operation may be used to
 * distinguish these two cases.
 *
 * @see #put(Object, Object)
 */
public V get(Object key) {
    //如果 key为 null,则直接去 table[0]处去检索即可。
    if (key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}
```

在这里如果key为null的话直接去检索table[0]处即可，下面我们来看看getEntry方法：

```java
/**
 * Returns the entry associated with the specified key in the
 * HashMap.  Returns null if the HashMap contains no mapping
 * for the key.
 */
final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }

    // 通过 key的 hashcode值计算 hash值
    int hash = (key == null) ? 0 : hash(key);
    // indexFor(hash&length-1) 获取最终数组索引，然后遍历链表，通过 equals方法比对找出对应记录
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

get方法的实现相对简单，key(hashcode)-->hash-->indexFor-->最终索引位置，找到对应位置table[i]，再查看是否有链表，遍历链表，通过key的equals方法比对查找对应的记录。要注意的是，有人觉得上面在定位到数组位置之后然后遍历链表的时候，e.hash == hash这个判断没必要，仅通过equals判断就可以。其实不然，试想一下，如果传入的key对象重写了equals方法却没有重写hashCode，而恰巧此对象定位到这个数组位置，如果仅仅用equals判断可能是相等的，但其hashCode和当前对象不一致，这种情况，根据Object的hashCode的约定，不能返回当前对象，而应该返回null。至于这里为何要这样后续会继续分析。

## <span id='3'>三、为何HashMap的数组长度一定是2的次幂？</span>
上面已经大概说了一下数组长度为何一定是2次幂的原因，下面这里我们重点分析一下。

我们在这里先分析一下上面提到的resize方法

```java
/**
 * Rehashes the contents of this map into a new array with a
 * larger capacity.  This method is called automatically when the
 * number of keys in this map reaches its threshold.
 *
 * If current capacity is MAXIMUM_CAPACITY, this method does not
 * resize the map, but sets threshold to Integer.MAX_VALUE.
 * This has the effect of preventing future calls.
 *
 * @param newCapacity the new capacity, MUST be a power of two;
 *        must be greater than current capacity unless current
 *        capacity is MAXIMUM_CAPACITY (in which case value
 *        is irrelevant).
 */
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

如果数组进行扩容，数组长度发生变化，而存储位置 index=h&(length-1),index也可能会发生变化，需要重新计算index，这里可能还不太明确，我们接着分析一下transfer方法：

```java
/**
 * Transfers all entries from current table to newTable.
 */
// 转移所有的元素到新的数组当中
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

这个方法将老数组中的数据逐个链表地遍历，扔到新的扩容后的数组中，我们的数组索引位置的计算是通过 对key值的hashcode进行hash扰乱运算后，再通过和 length-1进行位运算得到最终数组索引位置。

到这里我们可以知道，HashMap的数组长度一定要保持是2的次幂，例如长度为16的数组，二进制表示为10000，那么length-1就是15，二进制表示为01111，如果进行扩容后，长度变为32，二进制表示为100000，length-1为31，二进制表示为011111。如下面表示出来的情况，我们也可以看到这样子可以保证低位全为1，而扩容后仅有1位有差异，也就是多了最左位的1，这样在h&(length-1)时,只要h对应的最左边的那一个差异位为0，就能保证得到的新的数组索引和老数组索引一致，达到尽可能保证计算得到的位置不变，保证已散列良好的对象不需重新排位。

```
        0 1 1 1 1    length-1=15
      0 1 1 1 1 1    length-1=31
                     &
    0 0 0 0 1 0 0    h=4 
```

数组长度保持2的次幂，length-1的低位都为1，会使得获得的数组索引index散列更加均匀，如

```
    0 1 1 1 1 1 1    length-1=63
 &  1 0 1 0 1 0 0    h=84 
-----------------  
    0 0 1 0 1 0 0    index=20
```

从上面我们可以看出，h的高位在这里的与运算里并不会对结果产生影响，hash函数采用各种位运算可能也是为了使得低位更加散列均匀。

在这里我们只需要关注低位bit，如果低位全部为1，那么对于h低位部分来说，任何一位的变化都会对结果产生影响，也就是说，要得到index=20这个存储位置，h的低位有且只有这一种组合。

假设，如果数组长度length不是2次幂的话，如果为了得出index=20这个位置可能会出现这种情况

```
    0 1 1 1 1 0 1    length-1=61
 &  1 0 1 0 1 1 0    h=86 
-----------------  
    0 0 1 0 1 0 0    index=20
```

是不是看出了不同的地方？是的，这里能够计算得到index=20的组合就多了一种，这样子会使得碰撞的可能性变得更大，同时，index的那个位bit也就永远不可能为1，也就会导致部分空间是永远不可能存值导致白白浪费了空间。这也就是为什么数组长度必须为2的次幂的原因了。

## <span id='4'>四、重写equals方法需同时重写hashCode方法</span>
上面在get方法分析当中我们了解到了为何在遍历链表时需要e.hash == hash这一步操作，现在我们来详细分析一下原因，并且举个例子来说明一下只重写equals方法而不重写hashCode方法会发生什么问题，demo代码如下：

```java
/**
 * Override equals function Demo
 * 
 * @author WJR 2019年8月11日
 *
 */
public class Test2 {

	static class Person {

		private String id;
		private String name;

		public Person() {
			super();
		}

		public Person(String id, String name) {
			this.id = id;
			this.name = name;
		}

		public String getId() {
			return id;
		}

		public void setId(String id) {
			this.id = id;
		}

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		@Override
		public String toString() {
			return "Person [id=" + id + ", name=" + name + "]";
		}

		@Override
		public boolean equals(Object obj) {
			if (this == obj) {
				return true;
			}
			if (obj == null || getClass() != obj.getClass()) {
				return false;
			}
			Person person = (Person) obj;
			// 两个对象的id一致就可认为两者是同个对象
			return this.id == person.getId();
		}

	}

	public static void main(String[] args) {
		HashMap<Person, String> hashMap = new HashMap<Person, String>();
		Person person1 = new Person("1", "wjr");
		Person person2 = new Person("1", "wjrabc");
		// 将person1往hashMap中put
		hashMap.put(person1, "my-nickName");
		// 判断一下person1和person2，得到结果认为是同一个对象
		System.out.println("是否相等对象？" + person1.equals(person2));
		// 用person2作为key值取出，从逻辑上来说应该能得到my-nickName这个结果
		System.out.println("result: " + hashMap.get(person2));
	}

}
```

实际执行结果：

```
是否相等对象？true
result: null
```

如果我们已经对HashMap的原理有了一定了解，这个结果就不难理解了。尽管我们在进行get和put操作的时候，使用的key从逻辑上讲是等值的（通过equals比较是相等的），但由于没有重写hashCode方法，所以put操作时，key(hashcode1)-->hash-->indexFor-->最终索引位置 ，而通过key取出value的时候 key(hashcode1)-->hash-->indexFor-->最终索引位置，由于hashcode1不等于hashcode2，导致没有定位到一个数组位置而返回逻辑上错误的值null（也有可能碰巧定位到一个数组位置，但是也会判断其entry的hash值是否相等，上面get方法中有提到。）

所以，在重写equals的方法的时候，必须注意重写hashCode方法，同时还要保证通过equals判断相等的两个对象，调用hashCode方法要返回同样的整数值。而如果equals判断不相等的两个对象，其hashCode可以相同（只不过会发生哈希冲突，应尽量避免）。关于Object中的hashCode官方是这么说的，两个想等的对象，其hashCode必定相等。
