# 年轻代、老年代、永久代

在JVM的堆中，按代的划分如下：

![img-8](https://note.youdao.com/yws/api/personal/file/26A5B46282964120BB75EC2FE6235091?method=download&shareKey=575f9564d784085b2d9236c1c0dbc2c1)

**Young**：主要是用来存放新生的对象。

**Old**：主要存放应用程序中生命周期长的内存对象。

**Permanent**：是指内存的永久保存区域，主要存放Class和Meta的信息,Class在被 Load的时候被放入PermGen space区域. 它和存放Instance的Heap区域不同,GC(Garbage Collection)不会在主程序运行期对PermGen space进行清理，所以如果你的APP会LOAD很多CLASS的话,就很可能出现PermGen space错误。

# GC和Full GC的区别

GC（或Minor GC）：收集生命周期短的区域(Young area)。

Full GC（或Major GC）：收集生命周期短的区域(Young area)和生命周期比较长的区域(Old area)**对整个堆进行垃圾收集**。

