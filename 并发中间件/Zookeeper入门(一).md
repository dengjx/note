# 前言

## 分布式系统定义

> A distributed system is de ned as a software system that is composed of independent computing entities linked together by a computer network whose components communicate and coordinate with each other to achieve a common goal.
> 分布式系统是由独立的计算机通过网络连接在一起，并且通过一些组件来相互交流和协作来完成一个共同的目标。

想要更好的判断是否为好的分布式系统，可以看这些特性：

+ **资源共享**，例如存储空间，计算能力，数据，和服务等等
+ **扩展性**，从软件和硬件上增加系统的规模
+ **并发性** 多个用户同时访问
+ **性能** 确保当负载增加的时候，系统想要时间不会有影响
+ **容错性** 尽管一些组件暂时不可用了，整个系统仍然是可用的
+ **API抽象** 系统的独立组件对用户隐藏，仅仅暴露服务

## 分布式系统的难点

可以想象，假如一台计算机的出错概率为0.1%，那么1000台服务器的出错概率呢？一旦计算机的数量增多，出错的概率就大大的增加。

1. 多个相互独立的计算机，假设集群的配置信息在某个Master节点上，其余的节点从Master节点下载配置信息。假如Master节点挂了呢？假设Master节点是故障冗余的，但是配置信息是动态的传递给所有的其余节点的，而不是直接传过去。所有节点之间的信息如何保证一致呢？
2. 服务发现的问题，为了增加系统的可靠性，我们一般会在系统中增加更多的服务器。让其它机器知道新加入的节点在集群中的关系和服务，这个设计也需要非常周到的考虑
3. 机器数目众多，更容易出现机器故障，软件崩溃，网络延迟，拓扑改变等等，而这些类型的错误没有规律可循，因此在分布式系统，想实现高容错性是很难的。

为了解决这一系列问题，Zookeeper应运而生。

# Zookeeper介绍

&emsp;&emsp;ZooKeeper 是一个开源的分布式协调服务，由雅虎创建，是 Google Chubby的开源实现。分布式应用程序可以基于 ZooKeeper 实现诸如**数据发布/订阅**、**负载均衡**、**命名服务**、**分布式协调/通知**、**集群管理**、**Master 选举**、**配置维护**，**名字服务**、**分布式同步**、**分布式锁和分布式队列**等功能。其在一致性方面具有**顺序一致性、原子性、单一视图（每个client看到的数据一致）、可靠性、实时性**的特点。

**数据模型：**ZooKeeper 允许分布式进程通过共享的层次结构命名空间进行相互协调，这与标准文件系统类似。名称空间由 ZooKeeper 中的数据寄存器组成，称为 **Znode**，这些类似于文件和目录。与典型文件系统不同，ZooKeeper 数据保存在内存中，这意味着 ZooKeeper 可以实现高吞吐量和低延迟。

**顺序访问：**对于来自客户端的每个更新请求，ZooKeeper 都会分配一个全局唯一的递增编号。这个编号反应了所有事务操作的先后顺序，应用程序可以使用 ZooKeeper 这个特性来实现更高层次的同步原语。这个编号也叫做时间戳—zxid（ZooKeeper Transaction Id）。

**可构建集群：**为了保证高可用，最好是以集群形态来部署 ZooKeeper，这样只要集群中大部分机器是可用的（能够容忍一定的机器故障），那么 ZooKeeper 本身仍然是可用的。客户端在使用 ZooKeeper 时，需要知道集群机器列表，通过与集群中的某一台机器建立 TCP 连接来使用服务。客户端使用这个 TCP 链接来发送请求、获取结果、获取监听事件以及发送心跳包。如果这个连接异常断开了，客户端可以连接到另外的机器上。

# Zookeeper可以做什么？

## Zookeeper是什么

&emsp;&emsp;官方说辞：Zookeeper 分布式服务框架是Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：**统一命名服务**、**状态同步服务**、**集群管理**、**分布式应用配置项的管理**等。

&emsp;&emsp;这里看不懂是吧，没关系，我们看看Zookeeper提供了什么样的功能，看看这些功能能够提供什么。

## Zookeeper提供了什么

Zookeeper=文件系统+通知机制

### 1. 文件系统

Zookeeper维护一个类似文件系统的数据结构：

![img-1](https://note.youdao.com/yws/api/personal/file/CA25A5EC4E1E4B0DB5BC860CF31F5F01?method=download&shareKey=43cc42b366c7114dc4d4d42d890b9c7c)

&emsp;&emsp;每个子目录项如 NameService 都被称作为 **Znode**，和文件系统一样，我们能够自由的增加、删除Znode，在一个Znode下增加、删除子Znode，唯一的不同在于Znode是可以存储数据的。

有四种类型的Znode：

+ **PERSISTENT-持久化目录节点**

客户端与zookeeper断开连接后，该节点依旧存在

+ **PERSISTENT_SEQUENTIAL-持久化顺序编号目录节点**

客户端与zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号

+ **EPHEMERAL-临时目录节点**

客户端与zookeeper断开连接后，该节点被删除

+ **EPHEMERAL_SEQUENTIAL-临时顺序编号目录节点**

客户端与zookeeper断开连接后，该节点被删除，只是Zookeeper给该节点名称进行顺序编号

### 2. 通知机制

&emsp;&emsp;客户端注册监听它关心的目录节点，当目录节点发生变化（数据改变、被删除、子目录节点增加删除）时，zookeeper会通知客户端。 

## 能用Zookeeper做什么

### 1. 命名服务

&emsp;&emsp;在zookeeper的文件系统里创建一个目录，即有唯一的path。在我们使用tborg无法确定上游程序的部署机器时即可与下游程序约定好path，通过path即能互相探索发现。

### 2. 配置管理

&emsp;&emsp;程序总是需要配置的，如果程序分散部署在多台机器上，要逐个改变配置就会变得困难。怎么做到配置管理呢？我们可以把这些配置全部放到Zookeeper上去，保存在 Zookeeper的某个目录节点中，然后所有相关应用程序对这个目录节点进行监听，一旦配置信息发生变化，每个应用程序就会收到 Zookeeper 的通知，然后从 Zookeeper 获取新的配置信息应用到系统中就好。

![img-2](https://note.youdao.com/yws/api/personal/file/06F5F9A0E2A548C9953FE8C3826DC950?method=download&shareKey=911f2fb45f473f278604bb5e47cd60da)

### 3. 集群管理

&emsp;&emsp;所谓集群管理就是这么两点：是否有机器退出和加入、选举master。

&emsp;&emsp;对于第一点，所有机器约定在父目录GroupMembers下创建临时目录节点，然后监听父目录节点的子节点变化消息。一旦有机器挂掉，该机器与 zookeeper的连接断开，其所创建的临时目录节点被删除，所有其他机器都收到通知：某个兄弟目录被删除，新机器加入也是类似，所有机器都收到通知：新兄弟目录加入。

&emsp;&emsp;对于第二点，所有机器创建临时顺序编号目录节点，每次选取编号最小的机器作为master。

### 4. 分布式锁

&emsp;&emsp;有了zookeeper的一致性文件系统，锁的问题变得容易。锁服务可以分为两类，一个是保持独占，另一个是控制时序。

&emsp;&emsp;对于第一类，我们将zookeeper上的一个znode看作是一把锁，通过createznode的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁。当用完删除掉自己创建的distribute_lock 节点就释放出锁。

&emsp;&emsp;对于第二类， /distribute_lock 已经预先存在，所有客户端在它下面创建临时顺序编号目录节点，和选master一样，编号最小的获得锁，用完删除释放锁。

### 5. 队列管理

可以管理两种类型的队列：

1. 同步队列，当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达。
2. 队列按照 FIFO 方式进行入队和出队操作。

第一类，在约定目录下创建临时目录节点，监听节点数目是否是我们要求的数目。 

第二类，和分布式锁服务中的控制时序场景基本原理一致，入列有编号，出列按编号。在特定的目录下创建PERSISTENT_SEQUENTIAL节点，创建成功时Watcher通知等待的队列，队列删除序列号最小的节点用以消费。此场景下Zookeeper的znode用于消息存储，znode存储的数据就是消息队列中的消息内容，SEQUENTIAL序列号就是消息的编号，按序取出即可。由于创建的节点是持久化的，所以不必担心队列消息的丢失问题。

## Zookeeper分布式与数据复制

Zookeeper作为一个集群提供一致的数据服务，自然，它要在**所有机器间**做数据复制。数据复制的好处：

1、容错：一个节点出错，不致于让整个系统停止工作，别的节点可以接管它的工作； 
2、提高系统的扩展能力 ：把负载分布到多个节点上，或者增加节点来提高系统的负载能力； 
3、提高性能：让**客户端本地访问就近的节点，提高用户访问速度**。

从客户端读写访问的透明度来看，数据复制集群系统分下面两种： 

1、**写主(WriteMaster)**
对数据的修改提交给指定的节点。读无此限制，可以读取任何一个节点。这种情况下客户端需要对读与写进行区别，俗称读写分离；

2、**写任意(Write Any)**
对数据的修改可提交给任意的节点，跟读一样。这种情况下，客户端对集群节点的角色与变化透明。

对zookeeper来说，它采用的方式是**写任意**。通过增加机器，它的读吞吐能力和响应能力扩展性非常好，而写，随着机器的增多吞吐能力肯定下降（这也是它建立observer的原因），而响应能力则取决于具体实现方式，是**延迟复制保持最终一致性**，还是**立即复制快速响应**。

# Zookeeper基本概念

## 角色

Zookeeper中的角色主要有以下三类，如下表所示：

![img-3](https://note.youdao.com/yws/api/personal/file/DEC9E0A5F5594DF0855540B07D10E996?method=download&shareKey=e4f2d696bde2eacde5d5f6d61d329b52)

系统模型如图所示：

![img-4](https://note.youdao.com/yws/api/personal/file/C957831807F04AF5891A673D7D30E2D8?method=download&shareKey=7c4ae8f8fa1627728ae4dbedfd938ea0)

## 设计目的

1. **最终一致性：**client不论连接到哪个Server，展示给它都是同一个视图，这是zookeeper最重要的性能。

2. **可靠性：**具有简单、健壮、良好的性能，如果消息m被到一台服务器接受，那么它将被所有的服务器接受。

3. **实时性：**Zookeeper保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。但由于网络延时等原因，Zookeeper不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync()接口。

4. **等待无关（wait-free）：**慢的或者失效的client不得干预快速的client的请求，使得每个client都能有效的等待。

5. **原子性：**更新只能成功或者失败，没有中间状态。

6. **顺序性：**包括全局有序和偏序两种：全局有序是指如果在一台服务器上消息a在消息b前发布，则在所有Server上消息a都将在消息b前被发布；偏序是指如果一个消息b在消息a后被同一个发送者发布，a必将排在b前面。

## Zookeeper工作原理

&emsp;&emsp;Zookeeper 的核心是**原子广播**，这个机制保证了**各个Server之间的同步**。实现这个机制的协议叫做**Zab协议**。Zab协议有两种模式，它们分别是**恢复模式（选主）**和**广播模式（同步）**。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和 leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。

### Zookeeper如何保证事务的顺序一致性

&emsp;&emsp;zookeeper采用了**递增的事务Id**来标识，所有的proposal（提议）都在被提出的时候加上了zxid，zxid实际上是一个64位的数字，高32位是epoch（时期; 纪元; 世; 新时代）用来标识leader是否发生改变，如果有新的leader产生出来，epoch会自增，**低32位用来递增计数**。当新产生proposal的时候，会依据数据库的两阶段过程，首先会向其他的server发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么就会开始执行。

每个Server在工作过程中有三种状态：

- **LOOKING：**当前Server不知道leader是谁，正在搜寻
- **LEADING：**当前Server即为选举出来的leader
- **FOLLOWING：**leader已经选举出来，当前Server与之同步

# 参考

> zookeeper面试题 https://segmentfault.com/a/1190000014479433
>
> 我们能用zookeeper做什么 https://blog.csdn.net/zhangzq86/article/details/80981234
>
> Eureka的工作原理以及它与ZooKeeper的区别 https://www.cnblogs.com/snowjeblog/p/8821325.html
>
> Eureka和ZooKeeper的区别 https://blog.csdn.net/java_xth/article/details/82621776
>
> ZooKeeper学习 https://segmentfault.com/a/1190000015609181
>
> ZooKeeper快速入门 https://cloud.tencent.com/developer/article/1028715
>
> zookeeper入门到实战 https://blog.51cto.com/14230003/2374617
>
> ZooKeeper入门教程（一） https://www.jianshu.com/p/1f4c70d7ef40
>
> Zookeeper入门看这篇就够了 https://blog.csdn.net/java_66666/article/details/81015302
>
> zookeeper快速入门 https://www.cnblogs.com/niechen/p/8597344.html#_label3
>
> Zookeeper知识点整理 https://segmentfault.com/a/1190000012730375
>
> 【码农翻身】什么是ZooKeeper https://www.jianshu.com/p/06dfc108468d

