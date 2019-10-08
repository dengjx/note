# 概述

一个对象本身的内在结构需要一种描述方式，这个描述信息是以字节码的方法存储在方法区中的。
Class 本身就是一个对象，都以 KB 为单位，如果 new Integer() 为了表示一个数据就占用KB级别的内存就有点不值了，下面讲解 JVM 是如何做的。
为了表示对象的属性、方法等信息，不得不需要结构描述。Hotspot VM 使用对象头部的一个指针指向 Class 区域的方式来找到对象的 Class 描述，以及内部的方法、属性入口。如下图所示：

![img-10](https://note.youdao.com/yws/api/personal/file/C756138E79C54E64B73B02FDB7FBAD9F?method=download&shareKey=ecbf262dffadcc1c7253b4d4ba524f23)

在 HotSpot 虚拟机中，对象在内存中存储布局分为 3 块区域：对象头（Header）、实例数据（Instance Data）、对齐填充（Padding），下面详细讲解各部分内容。

# 对象头（Header）

HotSpot 虚拟机的对象头包括两部分（非数组对象）信息，如下图所示：

![img-11](https://note.youdao.com/yws/api/personal/file/604344F69ABF4D9EACC0887D6236A4F7?method=download&shareKey=e770b499df06a47cb18f420142f62256)

- 第一部分用于存储对象自身的**运行时数据**，如哈希码（HashCode）、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳、对象分代年龄，这部分信息称为“Mark Word”；Mark Word 被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据自己的状态复用自己的存储空间。
- 第二部分是**类型指针**，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例；
- 如果对象是一个 Java 数组，那在对象头中还必须有一块用于记录数组长度的数据。因为虚拟机可以通过普通 Java 对象的元数据信息确定 Java 对象的大小，但是从数组的元数据中无法确定数组的大小。

这部分数据的长度在 32 位和 64 位的虚拟机（未开启压缩指针）中分别为 32bit 和 64bit。

例如，在 32 位的 HotSpot 虚拟机中，如果对象处于未被锁定的状态下，那么 Mark Word 的 32bit 空间中的 25bit 用于存储对象哈希码，4bit 用于存储对象分代年龄，2bit 用于存储锁标志位，1bit 固定为 0，如下表所示：

![img-12](https://note.youdao.com/yws/api/personal/file/B137EB3101B249B9B57CB887EB101263?method=download&shareKey=12c781c44ec867779913a52ddba3c671)

在 32 位系统下，存放 Class 指针的空间大小是 4 字节，Mark Word 空间大小也是4字节，因此头部就是 8 字节，如果是数组就需要再加 4 字节表示数组的长度，如下表所示：

![img-13](https://note.youdao.com/yws/api/personal/file/0D7BEB2BE6FA4E499EA5524B23D85766?method=download&shareKey=ea1dd219fbf1b31f6c4901bb66a69492)

在 64 位系统及 64 位 JVM 下，开启指针压缩，那么头部存放 Class 指针的空间大小还是4字节，而 Mark Word 区域会变大，变成 8 字节，也就是头部最少为 12 字节，如下表所示：

![img-14](https://note.youdao.com/yws/api/personal/file/C2BA536A7A7B4F8883ED22D484EA2F1E?method=download&shareKey=6f4a43109f0f0264709aa742838d02da)

> 压缩指针：开启指针压缩使用算法开销带来内存节约，Java 对象都是以 8 字节对齐的，也就是以 8 字节为内存访问的基本单元，那么在地理处理上，就有 3 个位是空闲的，这 3 个位可以用来虚拟，利用 32 位的地址指针原本最多只能寻址 4GB，但是加上 3 个位的 8 种内部运算，就可以变化出 32GB 的寻址。

# 实例数据（Instance Data） 

实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。

这部分的存储顺序会受到虚拟机分配策略参数（FieldsAllocationStyle）和字段在 Java 源码中定义顺序的影响。

# 对齐填充（Padding）

对齐填充不是必然存在的，没有特别的含义，它仅起到占位符的作用。

由于 HotSpot VM 的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，也就是说对象的大小必须是 8 字节的整数倍。对象头部分是 8 字节的倍数，所以当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

# 估算对象大小

32 位系统下，当使用 new Object() 时，JVM 将会分配 8（Mark Word+类型指针） 字节的空间，128 个 Object 对象将占用 1KB 的空间。

如果是 new Integer()，那么对象里还有一个 int 值，其占用 4 字节，这个对象也就是 8+4=12 字节，对齐后，该对象就是 16 字节。

以上只是一些简单的对象，那么对象的内部属性是怎么排布的？

```java
Class A {
    int i;
    byte b;
    String str;
}
```

其中对象头部占用 ‘Mark Word’4 + ‘类型指针’4 = 8 字节；byte 8 位长，占用 1 字节；int 32 位长，占用 4 字节；String 只有引用，占用 4 字节；那么对象 A 一共占用了 8+1+4+4=17 字节，按照 8 字节对齐原则，对象大小也就是 24 字节。

这个计算看起来是没有问题的，对象的大小也确实是 24 字节，但是对齐（padding）的位置并不对：

在 HotSpot VM 中，对象排布时，间隙是在 4 字节基础上的（在 32 位和 64 位压缩模式下），上述例子中，int 后面的 byte，空隙只剩下 3 字节，接下来的 String 对象引用需要 4 字节来存放，因此 byte 和对象引用之间就会有 3 字节对齐，对象引用排布后，最后会有 4 字节对齐，因此结果上依然是 7 字节对齐。此时对象的结构示意图，如下图所示：

![img-15](https://note.youdao.com/yws/api/personal/file/3B2B1B7EC8B542F7B6FB3178EE260E9F?method=download&shareKey=c9da55da1a48e6212615e697d08bf42c)

# 参考

> [JVM——深入分析对象的内存布局](https://www.cnblogs.com/zhengbin/p/6490953.html)