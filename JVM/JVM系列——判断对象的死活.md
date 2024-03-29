# 一、引用计数法

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1，当引用失效时，计数器值就减1，任何时刻计数器为0的对象就是不可能再被使用的。
但是它很难解决对象之间相互循环引用的问题。
**比如说两个对象互相引用对方，导致它们的引用计数都不为0，于是引用计数算法无法通知GC收集器回收它们。**

# 二、可达性分析算法

&emsp;&emsp;目前主流实现中，都是通过该算法来判定对象是否存活的。这个算法基本思路就是通过一系列的成为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（在图论中叫，从GC Roots到这个对象“不可达”）时，则证明此对象是不可用的。

&emsp;&emsp;如图所示，对象o5、o6、o7虽然相互有关联，但是它们到GC Roots是不可达的，所以它们将会被判定为可回收的对象。

![img-9](https://note.youdao.com/yws/api/personal/file/544659FAC93942708EB4E709F6FA9E80?method=download&shareKey=e90fbb3e525dfd6e472c83f332088487)

在Java语言中，可作为GC Roots的对象包括下面几种：
1）虚拟机栈（栈帧中的本地变量表）中引用的对象。
2）方法区中类静态属性引用的对象。
3）方法区中常量引用的对象。
4）本地方法栈中JNI（即一般说的Native方法）引用的对象。