在Java虚拟机规范中将Java运行时数据划分为6种，分别为：

+ **PC寄存器（程序计数器）**
+ **Java栈**
+ **堆**
+ **方法区**
+ **运行时常量池**
+ **本地方法栈**

![img-1](https://note.youdao.com/yws/api/personal/file/CE043296E8B649F5B0353A091406833D?method=download&shareKey=a5f2580a61b40b8c5d6e29c58fa2745b)

# PC寄存器（程序计数器）
## 概念
什么是程序计数器？

<span style="color: red;">&emsp;&emsp;程序计数器是一个记录着当前线程所执行的字节码的行号指示器。</span>

&emsp;&emsp;我们知道，JAVA代码编译后的字节码在未经过JIT（实时编译器）编译前，其执行方式是通过“字节码解释器”进行解释执行。简单的工作原理为解释器读取装载入内存的字节码，按照顺序读取字节码指令。读取一个指令后，将该指令“翻译”成固定的操作，并根据这些操作进行分支、循环、跳转等流程。

&emsp;&emsp;上面的描述会让你有这么一种想法，假如没有程序计数器，JAVA代码也能够按着正常的指令顺序一步一步的执行下去，像正常的分支、循环等流程也能够正常跳转至相关指令顺序执行，这么一想，似乎没有必要存在程序计数器这么一个东西。其实不然，假如只有一条线程执行程序的时候，的确是完全按照顺序来执行指令，没有必要使用到程序计数器，但是，**实际上一个程序是由多个线程协同合作执行的**。

&emsp;&emsp;首先我们要搞清楚JVM的多线程实现方式：JVM的多线程是通过CPU时间片轮转（即线程轮流切换并分配处理器执行时间）算法来实现的。也就是说，当某一条线程在执行过程中，可能出现时间片耗尽而挂起，到下一条线程获取时间片执行，而当被挂起的线程重新获取时间片继续执行程序的时候，就需要知道在挂起之前执行到的指令是在哪一行或者说是哪个位置，而<span style="color: red;">在JVM中，通过程序计数器来记录某个线程的字节码执行位置</span>。因此，**程序计数器是具备线程隔离的特性**，也就是说，每个线程工作时都有属于自己的独立计数器。

总结一下：**PC寄存器（Program Counter Register）严格来说是一个数据结构，它用于保存当前正常执行的程序的内存地址。**

## 特点

+ **线程私有**：线程隔离性，每个线程工作时都有属于自己的独立计数器
+ 每个线程启动的时候，都会创建一个PC（Program Counter，程序计数器）寄存器。PC寄存器里保存有当前正在执行的JVM指令的地址
+ 每个线程都需要一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，这类内存区域为“线程私有”的内存
+ 如果线程正在执行的是一个Java方法，这个计数器记录的时正在执行的虚拟机字节码指令的地址；如果正在执行的时Native方法，这个计数器值则为空(Undefined)
+ 此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域

> 为何执行native方法时，计数器值为空呢？  
&emsp;&emsp;这是因为native方法是java通过JNI直接调用本地 C/C++ 库，可以近似的认为native方法相当于 C/C++ 暴露给java的一个接口，java通过调用这个接口从而调用到 C/C++ 方法。由于该方法是通过 C/C++ 而不是java进行实现。那么自然无法产生相应的字节码，并且 C/C++ 执行时的内存分配是由自己语言决定的，而不是由JVM决定的
![img-2](https://note.youdao.com/yws/api/personal/file/B28EFABD02D4431BA9205295F85F61F1?method=download&shareKey=bae7fddcfafd1d8d13e72146f9bd8d81)

# Java虚拟机栈
## 概念
<span style="color: red;">&emsp;&emsp;虚拟机栈是用于描述java方法执行的内存模型。</span>

&emsp;&emsp;每个java方法在执行时，会创建一个“栈帧（stack frame）”，栈帧的结构分为“局部变量表、操作数栈、动态链接、方法出口”几个部分（具体的作用会在字节码执行引擎章节中讲到，这里只需要了解栈帧是一个方法执行时所需要数据的结构）。我们常说的“堆内存、栈内存”中的“栈内存”指的便是虚拟机栈，确切地说，指的是虚拟机栈的栈帧中的局部变量表，因为这里存放了一个方法的所有局部变量。

&emsp;&emsp;方法调用时，创建栈帧，并压入虚拟机栈；方法执行完毕，栈帧出栈并被销毁，如下图所示：

![img-3](https://note.youdao.com/yws/api/personal/file/9037598C33B74F438784C965AEB52492?method=download&shareKey=d49d183484e30a83533fa9d68bd0b6b7)

&emsp;&emsp;在这个Java栈中又会含有多个栈帧，这些栈帧是与每个方法关联起来的，每运行一个方法就创建一个栈帧，每个栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机中入栈到出栈的过程。

&emsp;&emsp;<span style="color: red;">在编译程序代码的时候，栈帧中需要多大的局部变量表，多深的操作数栈都已经完全确定了，并且写入到方法表的Code属性之中，所以**一个栈帧需要分配多少内存，不会受程序运行期变量数据的影响**，而仅仅取决于具体的虚拟机实现。</span>

## 特点

+ 与程序计数器一样，**Java虚拟机栈也是线程私有的，它的生命周期与线程相同**。

## 虚拟机栈的StackOverflowError
&emsp;&emsp;在Java虚拟机规范中，如果单个线程请求的栈深度大于虚拟机所允许的深度，将抛出**StackOverflowError异常（栈溢出错误）**。

&emsp;&emsp;JVM会为每个线程的虚拟机栈分配一定的内存大小（-Xss参数），因此虚拟机栈能够容纳的栈帧数量是有限的，若栈帧不断进栈而不出栈，最终会导致当前线程虚拟机栈的内存空间耗尽，典型的一个无条件结束的递归函数调用，例子如下：

```java
/**
 * java栈溢出StackOverFlowError
 * JVM参数：-Xss128k
 */
public class JavaVMStackSOF {

    private int stackLength = -1;

    //通过递归调用造成StackOverFlowError
    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("Stack length:" + oom.stackLength);
            e.printStackTrace();
        }
    }

}
```

这里设置单个线程的虚拟机栈内存大小为128k，执行main方法后，抛出了StackOverflow异常

```
Stack length:983
java.lang.StackOverflowError
    at com.manayi.study.jvm.chapter2._02_JavaVMStackSOF.stackLeak(_02_JavaVMStackSOF.java:14)
    at com.manayi.study.jvm.chapter2._02_JavaVMStackSOF.stackLeak(_02_JavaVMStackSOF.java:15)
    at com.manayi.study.jvm.chapter2._02_JavaVMStackSOF.stackLeak(_02_JavaVMStackSOF.java:15)
    ······
```

## 虚拟机栈的OutOfMemoryError
&emsp;&emsp;如果虚拟机栈可以动态扩展（当前大部分的Java虚拟机都可以动态扩展，但Java虚拟机规范中也允许固定长度的虚拟机栈），如果扩展时无法申请到足够的内存，就会抛出OutOfMemoryError(OOM)异常。

&emsp;&emsp;JVM未提供设置整个虚拟机栈占用内存的配置参数。因为**操作系统分配给每个进程的内存是有限的，因此JVM虚拟机栈的内存并不是无限扩展下去的**，虚拟机栈的最大内存大致上等于“JVM进程能占用的最大内存（依赖于具体操作系统） - 最大堆内存 - 最大方法区内存 - 程序计数器内存（可以忽略不计） - JVM进程本身消耗内存”。

&emsp;&emsp;当虚拟机栈能够使用的最大内存被耗尽后，便会抛出OutOfMemoryError，可以通过不断开启新的线程来模拟这种异常，代码如下：

```java
/**
 * java栈溢出OutOfMemoryError
 * JVM参数：-Xss2m
 * 注意：运行该代码可能会导致操作系统假死！
 */
public class JavaVMStackOOM {

    private void dontStop() {
        while (true) {
        }
    }

    //通过不断的创建新的线程使Stack内存耗尽
    public void stackLeakByThread() {
        while (true) {
            Thread thread = new Thread(() -> dontStop());
            thread.start();
        }
    }

    public static void main(String[] args) {
        JavaVMStackOOM oom = new _03_JavaVMStackOOM();
        oom.stackLeakByThread();
    }

}
```

设置单个线程虚拟机栈的占用内存为2m并不断生成新的线程，最终虚拟机栈无法申请到新的内存，抛出异常：

```
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
```

PS：由于Java栈是与Java线程对应起来的，这个数据不是线程共享的，所以我们不用关心它的数据一致性问题，也不会存在同步锁的问题。

# 堆
## 概念
&emsp;&emsp;堆是存储Java对象的地方，它是JVM管理Java对象的核心存储区域，**在虚拟机启动时创建**。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。

&emsp;&emsp;堆是被**所有Java线程所共享的**，所以对它的访问需要注意同步问题，方法和对应的属性都需要保证一致性。

&emsp;&emsp;堆是用于存放对象的内存区域。因此，它是垃圾收集器（GC）管理的主要目标。因此也叫做GC堆。

## 特点
&emsp;&emsp;现在收集器基本都采用分代收集算法，所以Java堆中还可以细分为：**新生代（Eden空间、From Survivor和To Survivor空间）和老年代**。

+ 堆在逻辑上划分为“新生代”和“老年代”。由于JAVA中的对象大部分是朝生夕灭，还有一小部分能够长期的驻留在内存中，为了对这两种对象进行最有效的回收，将堆划分为新生代和老年代，并且执行不同的回收策略。不同的垃圾收集器对这2个逻辑区域的回收机制不尽相同
+ Java堆占用的内存可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像磁盘空间一样
+ 堆一般实现成可扩展内存大小，使用“-Xms”与“-Xmx”控制堆的最小与最大内存，扩展动作交由虚拟机执行。但由于该行为比较消耗性能，因此一般将堆的最大最小内存设为相等
+ 堆是所有线程共享的内存区域，因此每个线程都可以拿到堆上的同一个对象
+ **堆的生命周期是随着虚拟机的启动而创建**

## 堆异常
&emsp;&emsp;当堆无法分配对象内存且无法再扩展时，会抛出**OutOfMemoryError**异常。

&emsp;&emsp;一般来说，堆无法分配对象时会进行一次GC，如果GC后仍然无法分配对象，才会报内存耗尽的错误。可以通过不断生成新的对象但不释放引用来模拟这种情形：

```java
/**
 * java堆溢出demo
 * JVM参数：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {

    static class OOMObject {
    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        //不断创建新对象，使得Heap溢出
        while (true) {
            list.add(new OOMObject());
        }
    }

}
```

上述代码中对象不断的被创建而不进行引用释放，导致GC无法回收堆内存，最终OutOfMemoryError，错误信息：

```
java.lang.OutOfMemoryError: Java heap space
```

# 方法区
## 概念
&emsp;&emsp;方法区（也称非堆（Non-Heap））与Java堆一样，是各个**线程共享的内存区域**，它用于存储已被虚拟机加载的类信息、常量、静态变量、及时编译器编译后的代码等数据。

&emsp;&emsp;方法区中主要存储加载的类字节码、class/method/field等元数据对象、static-final常量、static变量、jit编译器编译后的代码等数据。

&emsp;&emsp;方法区中包含了一个特殊的区域“运行时常量池”，两者关系如下：

![img-4](https://note.youdao.com/yws/api/personal/file/F7ED6F7FB979439990966592B30228A4?method=download&shareKey=50528e1db083034bf2e3399f55ba6c8e)

> （1）加载的类字节码：要使用一个类，首先需要将其字节码加载到JVM的内存中。至于类的字节码来源，可以多种多样，如.class文件、网络传输、或cglib字节码框架直接生成。  
（2）class/method/field等元数据对象：字节码加载之后，JVM会根据其中的内容，为这个类生成Class/Method/Field等对象，它们用于描述一个类，通常在反射中用的比较多。不同于存储在堆中的java实例对象，这两种对象存储在方法区中。  
（3）static-final常量、static变量：对于这两种类型的类成员，JVM会在方法区为它们创建一份数据，因此同一个类的static修饰的类成员只有一份；  
（4）jit编译器的编译结果：以hotspot虚拟机为例，其在运行时会使用JIT即时编译器对热点代码进行优化，优化方式为将字节码编译成机器码。通常情况下，JVM使用“解释执行”的方式执行字节码，即JVM在读取到一个字节码指令时，会将其按照预先定好的规则执行栈操作，而栈操作会进一步映射为底层的机器操作；通过JIT编译后，执行的机器码会直接和底层机器打交道。如下图所示：  
![img-5](https://note.youdao.com/yws/api/personal/file/568BB8F87FBD48BCBFEFF6CDFBD9FDFC?method=download&shareKey=14ffccf3ea091aa29964db7d14935a64)

## 特点
&emsp;&emsp;方法区这个存储区域也属于后面介绍的Java堆中的一部分，也就是我们通常所说的Java堆中的永久区。

&emsp;&emsp;方法区这个区域有点特殊，由于它不像其他Java堆一样会频繁地被GC回收器回收，它存储的信息相对比较稳定，但是它仍然占用了Java堆的空间，所以仍然会被JVM的GC回收器来管理。

&emsp;&emsp;Java虚拟机规范对方法区的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集。

&emsp;&emsp;根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出**OutOfMemoryError异常**。

&emsp;&emsp;**永久代**是JDK1.8之前方法区的一种实现，用JVM内存，1.8之后方法区的实现改为元空间，直接使用JVM外内存。

> 在JDK1.7及之前的版本中，永久代保存在堆内存中，永久代不会被回收，如果操作不当，导致永久代中的数据量过大，那么这个时候程序会报出 OOM 问题。  
-XX:MaxPermSize: 设置永久代的最大值  
-XX:PremSize：设置永久代的初始大小  
注意！在JDK1.8中设置永久代会报错

```
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize10M; support was removed in 8.0
```

> JDK1.8之后取消了永久代，改变方法区的实现为元空间，功能依旧和永久代相似，唯一的区别是：永久代使用的是JVM的堆内存空间，而元空间使用的是物理内存，直接受到本机的物理内存限制。  
参数设置：  
-XX:MetaspaceSize：设置元空间的初始大小  
-XX:MaxMetaspaceSize：设置元空间的最大容量，默认是没有限制的（受本机物理内存限制）  
-XX:MinMetaspaceFreeRatio：执行GC后，最小的元空间剩余空间容量百分比，减小为分配空间所导致的垃圾收集  
-XX:MaxMetaspaceFreeRatio：执行GC后，最大的元空间剩余空间容量百分比，减小为释放空间所导致的垃圾收集

# 运行时常量池
&emsp;&emsp;运行时常量池（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译器生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。Java虚拟机规范没有对这部分做任何细节的要求。

&emsp;&emsp;运行时常量池相对于Class文件常量池的一个重要特性是**具备动态性**，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，**比如说String的intern()方法**。

```java
//使用StringBuilder在堆上创建字符串abc，再使用intern将其放入运行时常量池
String str = new StringBuilder("abc");
str.intern();
//直接使用字符串字面量xyz，其被放入运行时常量池
String str2 = "xyz";
```

## 方法区的实现

&emsp;&emsp;在上面已经有大概的说明了方法区在JVM中的实现方式：JDK1.8之前的是使用永久代，在JDK1.8及之后使用的是元空间。

&emsp;&emsp;可以思考一个问题，为什么使用“永久代”并将GC分代收集扩展至方法区这种实现方式不好，会导致OOM？首先要明白方法区的内存回收目标是什么，方法区存储了类的元数据信息和各种常量，它的内存回收目标理应当是对这些类型的卸载和常量的回收。但由于这些数据被类的实例引用，卸载条件变得复杂且严格，回收不当会导致堆中的类实例失去元数据信息和常量信息。因此，回收方法区内存不是一件简单高效的事情，往往GC在做无用功。另外随着应用规模的变大，各种框架的引入，尤其是使用了字节码生成技术的框架，会导致方法区内存占用越来越大，最终OOM。

&emsp;&emsp;在上面对于方法区实现的讨论中我们可以得知，无论哪种实现方式都会有一个最大的上限值，因此若方法区（含运行时常量池）占用内存到达其最大值，且无法再申请到内存时，便会抛出**OutOfMemoryError**。

&emsp;&emsp;下面的例子中，我们将使用cglib字节码生成框架不断生成新的类，最终使方法区内存占用满，抛出OutOfMemoryError：

```java
/**
 * java方法区溢出OutOfMemoryError（JVM参数适用于JDK1.6之前，借助CGLIB）
 * JVM参数：-XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class JavaMethodAreaOOM {

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback((MethodInterceptor) (o, method, objects, methodProxy) -> methodProxy.invokeSuper(objects, args));
            enhancer.create();
        }
    }

    static class OOMObject {
    }

}
```

报错信息为：

```
Caused by: java.lang.OutOfMemoryError: PermGen space
   at java.lang.ClassLoader.defineClass1(Native Method)
   ···
```

在日常开发中，不仅仅使cglib字节码生成框架会产生大量的class信息，动态语言、JSP、基于OSGI的应用都会在方法区额外产生大量的类信息。

# 本地方法栈

&emsp;&emsp;本地方法栈（Native Method Stack）与虚拟机栈所发挥的作用是非常相似的，它们之间的区别是：虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。

&emsp;&emsp;与虚拟机栈一样，本地方法栈区域也会抛出**StackOverflowError和OutOfMemoryError异常**和具有**线程隔离**的特点。

&emsp;&emsp;本地方法栈是为**JVM运行Native方法**准备的空间，它和前面介绍的Java栈的作用是类似的，由于很多Native方法都是用C语言实现的，所以它通常又叫C栈，除了在我们的代码中包含的常规的Native方法会使用这个存储空间，在JVM利用JIT技术时会将一些Java方法重新编译为Native Code代码，这些编译后的本地代码通常也是利用这个栈来跟踪方法的执行状态的。 

# 直接内存

&emsp;&emsp;直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。但是这部分内存也被频繁地使用，而且会导致OutOfMemoryError异常。为何会被频繁使用呢？原因是NIO这个包。

&emsp;&emsp;NIO（New input/output）是JDK1.4中新加入的类，引入了一种基于通道（channel）和缓冲区（buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过堆上的DirectByteBuffer对象对这块内存进行引用和操作。这样做显著提高了性能，避免在Java堆和Native堆中来回复制数据。

&emsp;&emsp;虽然这部分空间不会受到Java堆大小的限制，但是，因为是内存空间，所以会受到本机总内存大小以及处理器寻址空间的限制。

&emsp;&emsp;所以在配置虚拟机参数时，不能忽略直接内存，避免使各个内存区域总和大于物理内存限制。不然会导致动态扩展时出现OutOfMemoryError异常。

![img-7](https://note.youdao.com/yws/api/personal/file/924460FF42384569B41C8B5FB078986F?method=download&shareKey=dee516d26bc41f50d3f2bd0b4c829248)

&emsp;&emsp;上图可以看出，直接内存的大小并不受到 JVM 堆大小的限制，甚至不受到 JVM 进程内存大小的限制。它只受限于本机总内存（RAM及SWAP区或者分页文件）大小以及处理器寻址空间的限制（最常见的就是32位/64位CPU的最大寻址空间限制不同）。

## 直接内存的OufOfMemoryError

&emsp;&emsp;直接内存出现OutOfMemoryError的原因是对该区域进行内存分配时，其内存与其他内存加起来超过最大物理内存限制（包括物理的和操作系统级的限制），从而导致OutOfMemoryError。另外，若我们通过参数“-XX:MaxDirectMemorySize”指定了直接内存的最大值，其超过指定的最大值时，也会抛出内存溢出异常。

```java
/**
 * jvm直接内存溢出
 * JVM参数：-Xmx20M -XX:MaxDirectMemorySize=10M
 */
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws IllegalAccessException {
        //通过反射获取Unsafe类并通过其分配直接内存
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }

}
```

结果如下，可以看出，其抛出的内存溢出异常并没有指定是JVM那一块数据区域：

```
Exception in thread "main" java.lang.OutOfMemoryError
     at sun.misc.Unsafe.allocateMemory(Native Method)
     at com.manayi.study.jvm.chapter2._07_DirectMemoryOOM.main(_07_DirectMemoryOOM.java:22)
```

&emsp;&emsp;上面的例子有点特殊，因为我们使用到了Unsafe这个类（这个类为什么通过反射进行获取先不讨论），它的allocateMemory方法能够直接从堆外内存中申请内存（类比于c的malloc函数）。不同于DirectByteBuffer的内存分配方式（先计算是否有足够的可用内存再决定是手动抛异常还是向操作系统申请分配内存），Unsafe是直接向操作系统申请分配内存，若未申请到则抛异常。

# 总结

用一张图来总结各个区域存储的内容

![img-6](https://note.youdao.com/yws/api/personal/file/112588E37B5F404193FFA512B7BEF5C2?method=download&shareKey=34c0573d67ee2b04c25de20358a9309c)

# 参考

> [Java虚拟机（JVM）你只要看这一篇就够了！](https://blog.csdn.net/qq_41701956/article/details/81664921)  
>
> [《深入理解java虚拟机》读书笔记](https://www.cnblogs.com/manayi/category/1251410.html)  
>
> [[JVM内存结构——运行时数据区](https://www.cnblogs.com/zhengbin/p/5617023.html)](https://www.cnblogs.com/zhengbin/p/5617023.html)
>
> [深入JVM 原理（七）老年代、永久代和元空间](https://blog.csdn.net/qq_34707744/article/details/79288787)



