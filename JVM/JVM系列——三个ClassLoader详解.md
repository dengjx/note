# 简述

&emsp;&emsp;类装载工作由ClassLoader及其子类负责，ClassLoader是一个重要的Java执行时系统组件，它负责在运行时查找和装入Class字节码文件。JVM在运行时会产生三个ClassLoader：BootstrapClassLoader（根装载器）、ExtClassLoader（扩展类装载器）和AppClassLoader（系统类装载器）。其中，根装载器不是ClassLoader的子类，它使用C++编写，因此我们在Java中看不到它，根装载器负责装载JRE的核心类库，如JRE目标下的rt.jar、charsets.jar等。ExtClassLoader和AppClassLoader都是ClassLoader的子类。其中ExtClassLoader负责装载JRE目录ext中的JAR类包；AppClassLoader负责装载ClassPath路径下的类包。

- 启动类加载器（Bootstrap ClassLoader）：这个类加载器负责将存放在<JAVA_HOME>\lib目录中的。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，那直接使用null代替即可。
- 扩展类加载器（Extension ClassLoader）：这个加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载<JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。
- 应用程序类加载器（Application ClassLoader）：这个类加载器由sun.misc.Launcher$AppClassLoader实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义自己的类加载器，一般情况下这个就是程序中默认的类加载器。（PS：这个类加载器又称系统类加载器【System ClassLoader】）

&emsp;&emsp;我们的应用程序都是由这3种类加载器互相配合进行加载的，如果有必要，还可以加入自己定义的类加载器。这些类加载器之间的关系一般为：

![img-19](https://note.youdao.com/yws/api/personal/file/77A4BBE623424A76BE06F8B6C40E8220?method=download&shareKey=c7d9f5afcf98b277ace187021e362ba7)

&emsp;&emsp;上图展示的类加载器之间的这种层次关系，称为类加载器的**双亲委派模型**。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。这里类加载器之间的父子关系一般不会以继承的关系来实现，而是都使用组合关系来复用父加载器的代码。

> 关于**双亲委派模型**这个概念可以参考相关双亲委派模型markdown笔记。

这三个类装载器之间存在父子层级关系，即根装载器是ExtClassLoader的父装载器，ExtClassLoader是父类装载器。默认情况下，使用AppClassLoader装载应用程序的类，用以下代码证明：

```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        // Thread.currentThread():返回对当前正在执行的线程对象的引用
        // getContextClassLoader():返回该线程的上下文 ClassLoader
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        System.out.println("current loader:" + loader);
        System.out.println("parent loader:" + loader.getParent());
        System.out.println("grandparent loader:" + loader.getParent().getParent());
    }
}
```

结果会输出：

```
current loader:sun.misc.Launcher$AppClassLoader@1b6d3586
parent loader:sun.misc.Launcher$ExtClassLoader@1540e19d
grandparent loader:null   // 因为根类装载器在Java中访问不到，所有返回null
```

&emsp;&emsp;通过以上的输出信息，可以明白，ClassLoader是AppClassLoader，父ClassLoader是ExtClassLoader，祖父ClassLoader是根类装载器，因为在Java中无法获得它的句柄，所以直接返回null。

&emsp;&emsp;除了JVM默认的三个ClassLoader以外，可以编写自己的第三方类装载器，以实现一些特殊的需求。类文件被装载并解析后，在JVM内将拥有一个对应的java.lang.Class类描述对象，该类的实例都拥有指向这个类描述对象的引用，而类描述对象又拥有指向关联ClassLoader的引用。如下图所示：

![img-20](https://note.youdao.com/yws/api/personal/file/7630A5F600DB46E189F5931EADE3661D?method=download&shareKey=3970c29f68f6898cdef0965c51c8507a)

# ClassLoader重要方法

在Java中，ClassLoader是一个抽象类，位于java.lang包中。下面对该类的一些重要接口方法进行介绍：

- Class loadClass(String name)

　　name参数指定类装载器需要装载类的名字，必须使用全限定类名，如com.zhengbin.entity.Student。该方法有一个重载方法loadClass(String name, boolean resolve)，resolve参数告诉类装载器是否需要解析该类。在初始化类之前，应考虑进行类解析的工作，但并不是所有的类都需要解析，如果JVM只需要知道该类是否存在或找出该类的超类，那么就不需要进行解析。

- Class defineClass(String name, byte[] b, int off, int len)

&emsp;&emsp;将类文件的字节数组转换成JVM内部的java.lang.Class对象。字节数组可以从本地文件系统、远程网络获取。name为字节数组对应的全限定类名。

- Class findSystemClass(String name)

&emsp;&emsp;从本地文件系统载入Class文件，如果本地文件系统不存在该Class文件，将抛出ClassNotFoundException异常。该方法是JVM默认使用的装载机制。

- ClassLoader getParent()

&emsp;&emsp;获取类装载器的父装载器，除根装载器外，所有的类装载器都有且仅有一个父装载器，ExtClassLoader的父装载器是根装载器，因为根装载器非Java编写，所以无法获得，将返回null。

# 创建自定义类加载器

&emsp;&emsp;JVM中除根类加载器之外的所有类加载器都是ClassLoader子类的实例。可以通过扩展ClassLoader的子类，并重写该ClassLoader所包含的方法来实现自定义的类加载器，具体可以查看相关API文档。ClassLoader中包含了大量的protected方法，意味着这些方法都可以被子类重写。

&emsp;&emsp;ClassLoader类中有下面两个关键方法：

+ loadClass(String name, boolean resolve)

&emsp;&emsp;该方法为ClassLoader的入口点，根据指定名称来加载类，系统就是调用ClassLoader的该方法来获取指定类对应的Class对象。

+ findClass(String name)

&emsp;&emsp;根据指定名称来查找类。

如果需要实现自定义的ClassLoader，可以通过重写上面两个方法来实现，但是通常重写findClass方法，因为重写loadClass的逻辑更为复杂。

> loadClass方法的执行步骤如下：
>
> 1. 用findLoadedClass(String)来检查是否已经加载类，如果已经加载则直接返回。
> 2. 在父类加载器上调用loadClass方法。如果父类加载器为null，则使用根类加载器来加载。
> 3. 调用findClass(String)方法查找类。
>
> 从上面的执行步骤可以看出，重写findClass方法可以避免覆盖默认类加载器的父类委托、缓冲机制两种策略，如果重写loadClass方法的话就会导致实现逻辑复杂更多。

