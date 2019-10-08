# Kubernetes介绍

## 一、背景介绍

&emsp;&emsp;随着云计算的飞速发展，现在的应用普遍都是采取云化，云计算诞生了3种云服务模型：

- IaaS（基础架构即服务）
- PaaS（平台即服务）
- SaaS（软件即服务）

> 关于三种云服务模型详解可以自行网上查找，这里提供一篇博客参考
>
> 如何快速区分IaaS、PaaS 、SaaS？ https://cloud.tencent.com/developer/article/1339075

&emsp;&emsp;除了云计算的飞速发展，容器化技术也在突飞猛进，容器化技术（例如Docker）可以很好的在不中断服务的情况下方便开发人员维护及更新应用，容器或包含应用程序运行所需的所有内容的隔离环境使开发人员可以轻松地动态编辑和部署应用程序。因此，容器化已成为打包，部署和更新分布式Web应用程序的首选方法。但是，当分布式Web应用程序需要大规模维护和部署的时候，手动管理容器却是成为了运维人员或开发人员的负担，因此自动化部署容器及集群化管理所有容器的技术也飞速发展起来。

> 容器化技术带来的好处
>
> + 一次构建，到处运行
> + 容器的快速轻量
> + 完整的生态环境

## 二、什么是Kubernetes

&emsp;&emsp;Kubernetes(k8s)是Google开源的一款容器集群管理工具，是Google内部工具Borg的“开源版”。是一个全新的基于容器技术的分布式架构领先方案(k8s目前是公认的最先进的容器集群管理工具)，其在Docker技术的基础上，为容器化的应用提供部署运行、资源调度、服务发现和动态伸缩等一系列完整功能，提高了大规模容器集群管理的便捷性。

&emsp;&emsp;作为集群管理器出现的Borg系统，在其系统中运行着众多集群，而每个集群又由上万台服务器联接组成，Borg每时每刻都在处理来自众多应用程序所提交的成百上千的工作请求（Job）， Borg对这些工作请求（Job）进行接收、调度、启动，并对服务器上的容器进行启动、关闭、释放和监控。Borg论文中提到的三大优势:

- 为终端用户隐藏环境部署、资源管理和错误处理过程，终端用户仅需要关注应用的开发。
- 全局服务高可用、高可靠和性能监控。
- 自动化的将负载到成千上万的异构的服务器组成的集群中。

&emsp;&emsp;k8s对云环境中的资源进行了更高层次的抽象，通过将容器进行细致的组合，将最终的应用服务交给用户。

&emsp;&emsp;Kubernetes是一个完备的分布式系统支撑平台，具有完备的集群管理能力，多扩多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和发现机制、內建智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制以及多粒度的资源配额管理能力。同时Kubernetes提供完善的管理工具，涵盖了包括开发、部署测试、运维监控在内的各个环节。

### k8s的特点

使用Kubernetes可以快速高效地响应客户需求，其特点如下：

- 动态地对应用进行扩容。
- 无缝地发布和更新应用程序及服务。
- 按需分配资源以优化硬件使用。

Kubernetes的出现是为了减轻系统运维人员在公有云及私有云上编配和运行应用的负担。

Kubernetes从开发之初就致力于将其打造为简洁、可移植、可扩展且可自愈的系统平台，具体说明如下：

- 简洁：轻量级，简单，易上手
- 可移植：公有，私有，混合，多重云（multi-cloud）
- 可扩展: 模块化, 插件化, 可挂载, 可组合
- 可自愈: 自动布置, 自动重启, 自动复制
- 以应用程序为中心的管理： 将抽象级别从在虚拟硬件上运行操作系统上升到了在使用特定逻辑资源的操作系统上运行应用程序。这在提供了Paas的简洁性的同时拥有IssS的灵活性，并且相对于运行[12-factor](http://12factor.net/)应用程序有过之而无不及。
- 开发和运维的关注点分离: 提供构建和部署的分离；这样也就将应用从基础设施中解耦。
- 敏捷的应用创建和部署: 相对使用虚拟机镜像，容器镜像的创建更加轻巧高效。
- 持续开发，持续集成以及持续部署: 提供频繁可靠地构建和部署容器镜像的能力，同时可以快速简单地回滚(因为镜像是固化的)。
- 松耦合，分布式，弹性，自由的微服务: 应用被分割为若干独立的小型程序，可以被动态地部署和管理 -- 而不是一个运行在单机上的超级臃肿的大程序。
- 开发，测试，生产环境保持高度一致: 无论是再笔记本电脑还是服务器上，都采用相同方式运行。
- 兼容不同的云平台或操作系统上: 可运行与Ubuntu，RHEL，on-prem或者Google Container Engine，覆盖了开发，测试和生产的各种不同环境。
- 资源分离: 带来可预测的程序性能。
- 资源利用: 高性能，大容量。

### Kubernetes核心

&emsp;&emsp;Kubernetes中，Service是分布式集群架构的核心，一个Service对象拥有如下关键特征：

- 拥有一个唯一指定的名字
- 拥有一个虚拟IP（Cluster IP、Service IP、或VIP）和端口号
- 能够体统某种远程服务能力
- 被映射到了提供这种服务能力的一组容器应用上

&emsp;&emsp;Service的服务进程目前都是基于Socket通信方式对外提供服务，比如Redis、Memcache、MySQL、Web Server，或者是实现了某个具体业务的一个特定的TCP Server进程，虽然一个Service通常由多个相关的服务进程来提供服务，每个服务进程都有一个独立的Endpoint（IP+Port）访问点，但Kubernetes能够让我们通过服务连接到指定的Service上。有了Kubernetes内建的透明负载均衡和故障恢复机制，不管后端有多少服务进程，也不管某个服务进程是否会由于发生故障而重新部署到其他机器，都不会影响我们队服务的正常调用，更重要的是这个Service本身一旦创建就不会发生变化，意味着在Kubernetes集群中，我们不用为了服务的IP地址的变化问题而头疼了。

&emsp;&emsp;容器提供了强大的隔离功能，所有有必要把为Service提供服务的这组进程放入容器中进行隔离。为此，Kubernetes设计了Pod对象，将每个服务进程包装到相对应的Pod中，使其成为Pod中运行的一个容器。为了建立Service与Pod间的关联管理，Kubernetes给每个Pod贴上一个标签Label，比如运行MySQL的Pod贴上name=mysql标签，给运行PHP的Pod贴上name=php标签，然后给相应的Service定义标签选择器Label Selector，这样就能巧妙的解决了Service于Pod的关联问题。

&emsp;&emsp;在集群管理方面，Kubernetes将集群中的机器划分为一个Master节点和一群工作节点Node，其中，在Master节点运行着集群管理相关的一组进程kube-apiserver、kube-controller-manager和kube-scheduler，这些进程实现了整个集群的资源管理、Pod调度、弹性伸缩、安全控制、系统监控和纠错等管理能力，并且都是全自动完成的。Node作为集群中的工作节点，运行真正的应用程序，在Node上Kubernetes管理的最小运行单元是Pod。Node上运行着Kubernetes的kubelet、kube-proxy服务进程，这些服务进程负责Pod的创建、启动、监控、重启、销毁以及实现软件模式的负载均衡器。

&emsp;&emsp;在Kubernetes集群中，它解决了传统IT系统中服务扩容和升级的两大难题。你只需为需要扩容的Service关联的Pod创建一个Replication Controller简称（RC），则该Service的扩容及后续的升级等问题将迎刃而解。在一个RC定义文件中包括以下3个关键信息。

- 目标Pod的定义
- 目标Pod需要运行的副本数量（Replicas）
- 要监控的目标Pod标签（Label）

&emsp;&emsp;在创建好RC后，Kubernetes会通过RC中定义的的Label筛选出对应Pod实例并实时监控其状态和数量，如果实例数量少于定义的副本数量，则会根据RC中定义的Pod模板来创建一个新的Pod，然后将新Pod调度到合适的Node上启动运行，知道Pod实例的数量达到预定目标，这个过程完全是自动化。

### Kubernetes优势

+ 容器编排

+ 轻量级

+ 开源

+ 弹性伸缩

+ 负载均衡

### Kubernetes不是平台即服务（PaaS）

- Kubernetes并不对支持的应用程序类型有任何限制。 它并不指定应用框架，限制语言类型，也不仅仅迎合[12-factor](http://12factor.net/)模式. Kubernetes旨在支持各种多种多样的负载类型：只要一个程序能够在容器中运行，它就可以在Kubernetes中运行。
- Kubernetes并不关注代码到镜像领域。它并不负责应用程序的构建。不同的用户和项目对持续集成流程都有不同的需求和偏好，所以Kubernetes分层支持持续集成但并不规定和限制它的工作方式。
- 确实有不少PaaS系统运行在Kubernetes之上，比如[Openshift](https://github.com/openshift/origin)和[Deis](http://deis.io/)。同样你也可以将定制的PaaS系统，结合一个持续集成系统再Kubernetes上进行实施：只需生成容器镜像并通过Kubernetes部署。
- 由于Kubernetes运行再应用层而不是硬件层，所以它提供了一些一般PaaS提供的功能，比如部署，扩容，负载均衡，日志，监控，等等。无论如何，Kubernetes不是一个单一应用，所以这些解决方案都是可选可插拔的。

Kubernetes并不是单单的"编排系统"；它排除了对编排的需要:

- “编排”的技术定义为按照指定流程执行一系列动作：执行A，然后B，然后C。相反，Kubernetes有一系列控制进程组成，持续地控制从当前状态到指定状态的流转。无需关注你是如何从A到C：只需结果如此。这样将使得系统更加易用，强大，健壮和弹性。

## Kubernetes核心概念

&emsp;&emsp;Kubernetes组织结构主要是由**Node**、**Pod**、**Replication Controller**、**Deployment**、**Service**等多种资源对象组成的。其资源对象属性均保存在**etcd**提供的键值对存储库中，通过**kubectl**工具完成对资源对象的增、删、查、改等操作。我们可以将Kubernetes视为一个高度自动化的资源对象控制系统，它通过跟踪和对比**etcd**键值对库中保存的“对象原始信息”和当前环境中“对象实时信息”的差异来实现自动化控制和自动化纠错等功能的。

&emsp;&emsp;Kubernetes支持Docker和Rocket容器, 对其他的容器镜像格式和容器会在未来加入。

### Master

&emsp;&emsp;k8s集群的管理节点，负责管理集群，提供集群的资源数据访问入口。拥有Etcd存储服务（可选），运行Api Server进程，Controller Manager服务进程及Scheduler服务进程，关联工作节点Node。Kubernetes API server提供HTTP Rest接口的关键服务进程，是Kubernetes里所有资源的增、删、改、查等操作的唯一入口。也是集群控制的入口进程；Kubernetes Controller Manager是Kubernetes所有资源对象的自动化控制中心；Kubernetes Schedule是负责资源调度（Pod调度）的进程

&emsp;&emsp;Kubernetes中的Master是一台运行Kubernetes的主机，可以是物理机也可以是虚拟机，它是Kubernetes集群中的控制节点，它负责完成整个Kubernetes集群的创建、管理和控制，在Kubernetes集群中必不可少。

&emsp;&emsp;我们默认只能在Master上使用kubectl工具完成Kubernetes具体的操作命令，如果Master宕机或是离线，我们所有的控制命令都会失效。因此Master非常重要，在生产环境中，Master一般会配置高可用集群服务解决其单点故障问题。

Kubernetes 1.4 开始Master上的关键服务都是以Docker 容器实例的方式运行的，包括etcd服务。具体服务和其功能如下：

- Kubernetes API Server （ kube-apiserver），为kubernetes客户端提供HTTP Rest API接口的关键服务，是kubernetes中对象资源增、删、查、改等操作的唯一入口，同时也是kubernetes集群控制的入口。
- Kubernetes Controller Manager （ kube-controller-manager），为Kubernetes提供统一的自动化控制服务。
- Kubernetes Scheduler （kube-scheduler），为Kubernetes提供资源调度的服务，统一分配资源对象。
- Etcd Server（etcd），为Kubernetes保存所有对象的键值对数据信息。

### Node

&emsp;&emsp;在Kubernetes集群中，除了Master之外其它运行Kubernetes服务的主机称之为Node，在早期Kubernetes中称其为Minion。Node也可以是物理机或虚拟机。Node是Kubernetes集群中任务的负载节点，Master经过特殊设置后也可以作为Node负载任务。Kubernetes一般是由多个Node组成的，我们建议Node的数量是N+2的。当某个Node宕机后，其上的负载任务将会被Master自动转移到其它的Node上。之所以使用N+2的配置，是为了在Node意外宕机时Kubernetes集群的负载不会突然被撑满，导致性能急剧下降。

&emsp;&emsp;Node是Kubernetes集群架构中运行Pod的服务节点（亦叫agent或minion）。Node是Kubernetes集群操作的单元，用来承载被分配Pod的运行，是Pod运行的宿主机。关联Master管理节点，拥有名称和IP、系统资源信息。运行docker eninge服务，守护进程kunelet及负载均衡器kube-proxy.

- 每个Node节点都运行着以下一组关键进程
- kubelet：负责对Pod对于的容器的创建、启停等任务
- kube-proxy：实现Kubernetes Service的通信与负载均衡机制的重要组件
- Docker Engine（Docker）：Docker引擎，负责本机容器的创建和管理工作

&emsp;&emsp;Node节点可以在运行期间动态增加到Kubernetes集群中，默认情况下，kubelet会向master注册自己，这也是Kubernetes推荐的Node管理方式，kubelet进程会定时向Master汇报自身情报，如操作系统、Docker版本、CPU和内存，以及有哪些Pod在运行等等，这样Master可以获知每个Node节点的资源使用情况，冰实现高效均衡的资源调度策略。

&emsp;&emsp;Kubernetes 1.4 开始Node上的关键服务都是以Docker 实例的方式运行的，具体服务和功能如下：

- kubelet，负责Pod对应的容器实例的创建、启动、停止、删除等任务，接受Master传递的指令实现Kubernetes集群管理的基本功能。
- kube-proxy，实现kubernetes service功能和负载均衡机制的服务。

&emsp;&emsp;Node节点通过kubelet服务向Master注册，可以实现动态的在Kubernetes集群中添加和删除负载节点。已加入Kubernetes集群中的Node节点还会通过kubelet服务动态的向Master提交其自身的资源信息，例如主机操作系统、CPU、内存、Docker版本和网络情况。如果Master发现某Node超时未提交信息，Node会被判定为“离线”并标记为“不可用（Not Ready），随后Master会将此离线Node上原有Pod迁移到其它Node上。

### Pod

&emsp;&emsp;在Kubenetes中所有的容器均在Pod中运行，一个Pod可以承载一个或者多个相关的容器。同一个Pod中的容器会部署在同一个物理机器上并且能够共享资源。一个Pod也可以包含0个或者多个磁盘卷组（volumes），这些卷组将会以目录的形式提供给一个容器或者被所有Pod中的容器共享。对于用户创建的每个Pod，系统会自动选择那个健康并且有足够资源的机器，然后开始将相应的容器在那里启动，你可以认为Pod就是虚拟机。当容器创建失败的时候，容器会被节点代理（node agent）自动重启，这个节点代理（node agent）就是kubelet服务。在我们定义了副本控制器（replication controller）之后，如果Pod或者服务器故障的时候，容器会自动的转移并且启动。

&emsp;&emsp;Pod是Kubernetes的基本操作单元，把相关的一个或多个容器构成一个Pod，通常Pod里的容器运行相同的应用。Pod包含的容器运行在同一个物理机器上，看作一个统一管理单元，共享相同的卷（volumes）和网络名字空间（network namespace）、IP和端口（Port）空间。在Kubernetes集群中，一个Pod中的容器可以和另一个Node上的Pod容器直接通讯。

&emsp;&emsp;用户可以自己创建并管理Pod，但是Kubernetes可以极大的简化管理操作，它能让用户指派两个常见的跟Pod相关的活动：1) 基于相同的Pod配置，部署多个Pod副本；2）当一个Pod或者它所在的机器发生故障的时候创建替换的Pod。Kubernetes的API对象用来管理这些行为，我们将其称作副本控制器（Replication Controller），它用模板的形式定义了Pod，然后系统根据模板实例化出一些Pod。Pod的副本集合可以共同组成应用、微服务，或者在一个多层应用中的某一层。一旦Pod创建好，Kubernetes系统会持续的监控他们的健康状态，和它们运行时所在的机器的健康状况。如果一个Pod因为软件或者机器故障，副本控制器（Replication Controller）会自动在健康的机器上创建一个新的Pod，来保证pod的集合处于冗余状态。

&emsp;&emsp;Pod运行于Node节点上，若干相关容器的组合。Pod内包含的容器运行在同一宿主机上，使用相同的网络命名空间、IP地址和端口，能够通过localhost进行通。Pod是Kurbernetes进行创建、调度和管理的最小单位，它提供了比容器更高层次的抽象，使得部署和管理更加灵活。一个Pod可以包含一个容器或者多个相关容器。

&emsp;&emsp;Pod其实有两种类型：普通Pod和静态Pod，后者比较特殊，它并不存在Kubernetes的etcd存储中，而是存放在某个具体的Node上的一个具体文件中，并且只在此Node上启动。普通Pod一旦被创建，就会被放入etcd存储中，随后会被Kubernetes Master调度到摸个具体的Node上进行绑定，随后该Pod被对应的Node上的kubelet进程实例化成一组相关的Docker容器冰启动起来，在。在默认情况下，当Pod里的某个容器停止时，Kubernetes会自动检测到这个问起并且重启这个Pod（重启Pod里的所有容器），如果Pod所在的Node宕机，则会将这个Node上的所有Pod重新调度到其他节点上。

### Label

&emsp;&emsp;Label用来给Kubernetes中的对象分组。Label通过设置键值对（key-value）方式在创建Kubernetes对象的时候附属在对象之上。一个Kubernetes对象可以定义多个Labels（key=value），并且key和value均由用户自己指定，同一组Label（key=value）可以指定到多个Kubernetes对象，Label可以在创建Kubernetes对象时设置，也可以在对象创建后通过kubectl或kubernetes api添加或删除。其他Kubernetes对象可以通过标签选择器（Label Selector）选择作用对象。你可以把标签选择器（Lebel Selector）看作关系查询语言（SQL）语句中的where条件限定词。

&emsp;&emsp;Lable实现的是将指定的资源对象通过不同的Lable进行捆绑，灵活的实现多维度的资源分配、调度、配置和部署。

&emsp;&emsp;Kubernetes中的任意API对象都是通过Label进行标识，Label的实质是一系列的Key/Value键值对，其中key于value由用户自己指定。Label可以附加在各种资源对象上，如Node、Pod、Service、RC等，一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上去。Label是Replication Controller和Service运行的基础，二者通过Label来进行关联Node上运行的Pod。

&emsp;&emsp;我们可以通过给指定的资源对象捆绑一个或者多个不同的Label来实现多维度的资源分组管理功能，以便于灵活、方便的进行资源分配、调度、配置等管理工作。

一些常用的Label如下：

- 版本标签："release":"stable","release":"canary"......
- 环境标签："environment":"dev","environment":"qa","environment":"production"
- 架构标签："tier":"frontend","tier":"backend","tier":"middleware"
- 分区标签："partition":"customerA","partition":"customerB"
- 质量管控标签："track":"daily","track":"weekly"

&emsp;&emsp;Label相当于我们熟悉的标签，给某个资源对象定义一个Label就相当于给它大了一个标签，随后可以通过Label Selector（标签选择器）查询和筛选拥有某些Label的资源对象，Kubernetes通过这种方式实现了类似SQL的简单又通用的对象查询机制。

Label Selector在Kubernetes中重要使用场景如下:

- kube-Controller进程通过资源对象RC上定义Label Selector来筛选要监控的Pod副本的数量，从而实现副本数量始终符合预期设定的全自动控制流程
- kube-proxy进程通过Service的Label Selector来选择对应的Pod，自动建立起每个Service岛对应Pod的请求转发路由表，从而实现Service的智能负载均衡
- 通过对某些Node定义特定的Label，并且在Pod定义文件中使用Nodeselector这种标签调度策略，kuber-scheduler进程可以实现Pod”定向调度“的特性

### Replication Controller（RC）

&emsp;&emsp;Replication Controller确保任何时候Kubernetes集群中有指定数量的Pod副本(replicas)在运行， 如果少于指定数量的Pod副本(replicas)，Replication Controller会启动新的容器（Container），反之会杀死多余的以保证数量不变。Replication Controller使用预先定义的Pod模板创建pods，一旦创建成功，Pod模板和创建的Pod没有任何关联，可以修改Pod模板而不会对已创建Pod有任何影响。也可以直接更新通过Replication Controller创建的pod。对于利用pod模板创建的pod，Replication Controller根据标签选择器（Label Selector）来关联，通过修改Pod的Label可以删除对应的Pod。

Replication Controller的定义包含如下的部分：

- Pod的副本数目（Replicas）
- 用于筛选Pod的标签选择器（Label Selector）
- 用于创建Pod的标准配置模板（Template）

Replication Controller主要有如下用法：

- 编排（Rescheduling）：Replication Controller会确保Kubernetes集群中指定的pod副本(replicas)在运行， 即使在节点出错时。
- 缩放（Scaling）：通过修改Replication Controller的副本(replicas)数量来水平扩展或者缩小运行的pod。
- 滚动升级（Rolling updates）： Replication Controller的设计原则使得可以一个一个地替换Pod来滚动升级（rolling updates）服务。
- 多版本任务（Multiple release tracks）: 如果需要在系统中运行多版本（multiple release）的服务，Replication Controller使用Labels来区分多版本任务（multiple release tracks）。

&emsp;&emsp;由于Replication Controller 与Kubernetes API中的模块有同名冲突，所以从Kubernetes 1.2 开始并在Kubernetes 1.4 中它由另一个概念替换，这个新概念的名称为副本设置（Replica Set），Kubernetes官方将其称为”下一代RC“。Replicat Set 支持基于集合的Label Selector（set-based selector），而Replication Controller 仅仅支持基于键值对等式的Label Selector（equality-based selector）。此外，Replicat Set 在Kubernetes 1.4中也不再单独使用，它被更高层次的资源对象Deployment 使用，所以在Kubernetes 1.4中我们使用Deployment定义替换了之前的Replication Controller定义。

&emsp;&emsp;Replication Controller用来管理Pod的副本，保证集群中存在指定数量的Pod副本。集群中副本的数量大于指定数量，则会停止指定数量之外的多余容器数量，反之，则会启动少于指定数量个数的容器，保证数量不变。Replication Controller是实现弹性伸缩、动态扩容和滚动升级的核心。

### Deployment

&emsp;&emsp;在Kubernetes 1.2开始，定义了新的概念Deployment用以管理Pod和替换Replication Controller。

&emsp;&emsp;你可以认为Deployment是新一代的副本控制器。其工作方式和配置基本与Replication Controller差不多，后面我们主要使用的副本控制器是Deployment。

### Horizontal Pod Autoscaler （HPA）

&emsp;&emsp;Horizontal Pod Autoscaler 为Pod 横向自动扩容，简称HPA。Kubernetes可以通过RC或其替代对象监控Pod的负载变化情况，以此制定针对性方案调整目标Pod的副本数以增加其性能。

HPA使用以下两种方法度量Pod的指标：

- CPUUtilizationPercentage，目标Pod所有副本自身的CPU利用率的平均值
- 应用自定义度量指标，例如每秒请求数（TPS）

### Service

&emsp;&emsp;Kubernetes中Pod是可创建可销毁而且不可再生的。 Replication Controllers可以动态的创建、配置和销毁Pod。虽然我们可以设置Pod的IP，但是Pod的IP并不能得到稳定和持久化的保证。这将会导致一个凸出的问题，如果在Kubernetes集群中，有一些后端Pod（backends）为另一些前端Pod（frontend）提供服务或功能驱动，如何能保证前端（frontend）能够找到并且链接到后端（backends）。这就需要称之为服务（Service）的Kubernetes对象来完成任务了。

&emsp;&emsp;Services也是Kubernetes的基本操作单元，是Kubernetes运行用户应用服务的抽象，每一个服务后面都有很多对应的容器来支持，通过Proxy的port和服务selector决定服务请求传递给后端提供服务的容器，对外表现为一个单一访问接口，外部不需要了解后端如何运行，这给扩展或维护后端带来很大的好处。

&emsp;&emsp;Service定义了Pod的逻辑集合和访问该集合的策略，是真实服务的抽象。Service提供了一个统一的服务访问入口以及服务代理和发现机制，关联多个相同Label的Pod，用户不需要了解后台Pod是如何运行。

&emsp;&emsp;Kubernetes中的每个Service其实就是我们称之为“微服务”的东西。

Service同时还是Kubernetes分布式集群架构的核心，Service具有以下特性：

- 拥有唯一指定的名字
- 拥有虚拟IP
- 能够提供某种网络或Socket服务
- 能够将用户请求映射到提供服务的一组容器的应用上

&emsp;&emsp;一般情况下，Service通常是由多个Pod及相关服务进程来提供服务的，每个服务进程都具有一个独立的访问点（Endpoint 一组IP和端口），Kubernetes可以通过内建的透明负载均衡和故障恢复机制，排除突发故障并提供我们不间断的访问。

> 外部系统访问Service的问题
>
> 首先需要弄明白Kubernetes的三种IP这个问题：
>
> + Node IP：Node节点的IP地址
> + Pod IP： Pod的IP地址
> + Cluster IP：Service的IP地址
>
> &emsp;&emsp;首先,Node IP是Kubernetes集群中节点的物理网卡IP地址，所有属于这个网络的服务器之间都能通过这个网络直接通信。这也表明Kubernetes集群之外的节点访问Kubernetes集群之内的某个节点或者TCP/IP服务的时候，必须通过Node IP进行通信。
>
> &emsp;&emsp;其次，Pod IP是每个Pod的IP地址，他是Docker Engine根据docker0网桥的IP地址段进行分配的，通常是一个虚拟的二层网络。
>
> &emsp;&emsp;最后Cluster IP是一个虚拟的IP，但更像是一个伪造的IP网络，原因有以下几点：
>
> - Cluster IP仅仅作用于Kubernetes Service这个对象，并由Kubernetes管理和分配P地址
> - Cluster IP无法被ping，他没有一个“实体网络对象”来响应
> - Cluster IP只能结合Service Port组成一个具体的通信端口，单独的Cluster IP不具备通信的基础，并且他们属于Kubernetes集群这样一个封闭的空间。
>
> Kubernetes集群之内，Node IP网、Pod IP网于Cluster IP网之间的通信，采用的是Kubernetes自己设计的一种编程方式的特殊路由规则。

### Volume

&emsp;&emsp;Volume是Pod中能够被多个容器访问的共享目录。Kubernetes中的Volume定义的Pod上，然后被一个Pod里的多个容器挂载到具体的目录上，Volume的生命周期与Pod的生命周期相同，但并不是与Pod中的容器相关，当Pod中的容器终止或重启，Volume中的数据不会丢失。Kubernetes中的Volume支持GlusterFS和Ceph这样的分布式文件系统和本地EmptyDir和HostPath这样的本地文件系统。

### Persistent Volume

&emsp;&emsp;Persistent Volume 不同于前面提到的Volume ，Volume是分配给Pod的存储资源，而Persistent Volume是Kubernetes集群中的网络存储资源，我们可以在这个资源中划分子存储区域分配给Pod作为Volume使用。Persistent Volume 简称 PV，作为Pod 的Volume使用时，还需要分配Persistent Volume Claim 作为Volume，Persistent Volume Claim简称PVC。

Persistent Volume 有以下特点：

- Persistent Volume 只能是网络存储，并不挂接在任何Node，但可以在每个Node上访问到
- Persistent Volume 并不是第一在Pod上，而是独立于Pod定义到Kubernetes集群上
- Persistent Volume 目前支持的类型如下：NFS、RDB、iSCSI 、AWS ElasticBlockStore、GlusterFS和Ceph等

### Namespace

&emsp;&emsp;Namespace 是Linux 中资源管理的重要概念和应用，它实现了多用户和多进程的资源隔离。Kubernetes中将集群中的资源对象运行在不同的Namespace下，形成了相对独立的不同分组、类型和资源群。

&emsp;&emsp;在Kubernetes 集群启动后，会创建第一个默认Namespace 叫做”default“。用户在创建自有的Pod、RC、Deployment 和Service 指定运行的Namespace，在不明确指定的情况下默认使用”default“。

&emsp;&emsp;Kubernetes 集群的资源配额管理也是通过Namespace结合Linux 系统的Cgroup 来完成对不同用户Cpu使用量、内存使用量、磁盘访问速率等资源的分配。

