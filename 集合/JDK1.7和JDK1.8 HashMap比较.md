# HashMap
HashMap是基于哈希表的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。

下面，我们可以想想HashMap为何采用hash表存数据？

在一般的集合里面，集合是无序的，如果我们想要向这个集合中添加数据的时候，需要和这个集合中的所有数据进行 `equals` 比较，因此当这个集合中的数据量很大的时候，这个效率是非常低的。而使用 hashCode 来定位元素，这个是非常快的，因为使用 hash 表来存储数据是用的 hashCode 来计算存储位置并存放数据，而且 hash 表的底层是用的 `Entry` 对象数组来实现的，数组的查询只需要知道数据的下标就能定位到元素并进行修改或新增。

## JDK1.7和JDK1.8的HashMap数据结构比较

+ JDK1.7的底层数据结构是**数组+链表**的形式构成的，数组的单元是实现Map的Entry，链表的单元也是，Map的Entry是一组key、value值的节点。
+ JDK1.8的底层数据结构和JDK1.7相似，但是却又有点不同，因为JDK1.8新增加入了红黑树的数据结构，结合在一起的数据结构就是**数组+链表+红黑树**。

### 为何要加入红黑树这个数据结构呢？
为了解答这个问题，首先，我们先了解一下hashMap如何添加一个对象。

如下图，当往hashMap中put一个key、value键值对时，会触发hashMap中的hash算法计算得到相应存储在数组中的索引值，然后进行存放（hash算法用于计算得到键值对数据存放的位置）。
![img-1](https://note.youdao.com/yws/api/personal/file/7D363709A5B24AAA91B1461FF27AC9FC?method=download&shareKey=5b74952c9bf59644b94edf79eda571a0)

当再次往这个hashMap中put一个键值对时，根据hash算法计算会得到数组的索引值，然后根据索引值查找数组，判断数组在该索引值上是否存在对象，如果不存在对象则存入，如上图，若存在对象则通过equals比较两个对象的key值是否相等，若相等则覆盖value值，如下图。
![img-2](https://note.youdao.com/yws/api/personal/file/49FEB51A00C244A189ACB0D1A6DFE00E?method=download&shareKey=07fb15748dbf70c3d878580493571090)

如果比较两个对象中的key值不相等，就会形成链表结构。在JDK1.7中后加的在前面，先加的移下，这种情况称为hash冲突也叫碰撞。如下图。

![img-3](https://note.youdao.com/yws/api/personal/file/6D4FCE013BF64A49807AC90AE935BF43?method=download&shareKey=0e7f6331f941ee1b70b6cf0436493875)

这种碰撞现象应该要尽量避免，否则当其中一个索引下面的链表存的对象比较多的时候，这个时候插入一个对象还是在这个索引的时候，就需要这个索引下面的链表存放的对象进行逐个对比，了解链表结构的都知道，链表在新增删除节点是非常快的，而进行查询则会比较慢，而这种情况下，该索引下面的数据量大就需要一个个查询比较，这样效率是非常低下的，因为最差的情况下需要所有都遍历到才进行插入。

即使我们严谨的重写了 `equals` 方法和 `hashCode` 方法，这种碰撞现象也是无法避免的，因为数组的长度是有限的，所以HashMap提供了一个加载因子避免碰撞，默认是0.75。即当元素到达现有的hash表的75%时扩容，一旦扩充就会重新排序hash表，减少碰撞概率。

JDK1.7源码的碰撞因子：

```java
/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

但是，即使是使用了加载因子来避免碰撞，也是无法完全避免碰撞现象。为了解决这个问题，所以在JDK1.8里，为了减少这种碰撞现象出现，JDK1.8中的HashMap存储结构是由**数组+链表+红黑树**这三种数据结构形成（红黑树查询删除快新增慢）。存储结构下图所示，根据key的hash与table长度确定table位置，同一个位置的key以链表形式存储，超过一定限制链表转为树。数组的具体存取规则是 `tab[(n-1) & hash]`,其中tab为node数组，n为数组的长度，hash为key的hash值。

![img-4](https://note.youdao.com/yws/api/personal/file/8FAD76D2E66B49D188650C8137A019E2?method=download&shareKey=69ff14f1da1fd9310f9d13812b19a8ab)

转换列表为树的条件是链表中数据大于临界值8并且数组的size大于64就转换链表为树。

相关源码：

```java
/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2 and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
 * between resizing and treeification thresholds.
 */
static final int MIN_TREEIFY_CAPACITY = 64;

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) //这一步是初始化数组
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null) //这一步是保证计算出来的hash不会越界
        //存放数组的索引是 (数组长度 - 1) 和 hash 的与运算得出来的。
        // hash 的值是先经过key的hashCode的后16位与前16位进行异或操作得来
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //若链表容量大于TREEIFY_THRESHOLD则调用转化树treeifyBin方法
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

/**
 * Replaces all linked nodes in bin at index for given hash unless
 * table is too small, in which case resizes instead.
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果数组为空或者数组长度小于最小转化树容量64则进行resize扩容操作
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```





