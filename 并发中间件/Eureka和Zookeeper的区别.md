# Eureka简介

Eureka 是 Netflix 出品的用于实现服务注册和发现的工具。 Spring Cloud 集成了 Eureka，并提供了开箱即用的支持。其中， Eureka 又可细分为 Eureka Server 和 Eureka Client。

## 基本原理

![img-5](https://note.youdao.com/yws/api/personal/file/1CEDADE2C0234C588B972E294C045FA8?method=download&shareKey=907dfe1a06b03e7caca483ba5a689d83)

上图是Eureka的官方架构图，这是基于集群配置的Eureka：

+ 处于不同节点的eureka通过Replicate进行数据同步
+ Application Service为服务提供者
+ Application Client为服务消费者
+ Make Remote Call完成一次服务调用

&emsp;&emsp;服务启动后向Eureka注册，Eureka Server会将注册信息向其他Eureka Server进行同步，当服务消费者要调用服务提供者，则向服务注册中心获取服务提供者地址，然后会将服务提供者地址缓存在本地，下次再调用时，则直接从本地缓存中取，完成一次调用。

&emsp;&emsp;当服务注册中心Eureka Server检测到服务提供者因为宕机、网络原因不可用时，则在服务注册中心将服务置为`DOWN`状态，并把当前服务提供者状态向订阅者发布，订阅过的服务消费者更新本地缓存。

&emsp;&emsp;服务提供者在启动后，周期性（默认30秒）向Eureka Server发送心跳，以证明当前服务是可用状态。Eureka Server在一定的时间（默认90秒）未收到客户端的心跳，则认为服务宕机，注销该实例。

## Eureka自我保护机制

&emsp;&emsp;在默认配置中，Eureka Server在默认90s没有得到客户端的心跳，则注销该实例，但是往往因为微服务跨进程调用，网络通信往往会面临着各种问题，比如微服务状态正常，但是因为网络分区故障时，Eureka Server注销服务实例则会让大部分微服务不可用，这很危险，因为服务明明没有问题。

&emsp;&emsp;为了解决这个问题，Eureka 有自我保护机制，通过在Eureka Server配置如下参数，可启动保护机制

```properties
eureka.server.enable-self-preservation = true
```

&emsp;&emsp;它的原理是，当Eureka Server节点在短时间内丢失过多的客户端时（可能发送了网络故障），那么这个节点将进入自我保护模式，不再注销任何微服务，当网络故障回复后，该节点会自动退出自我保护模式。

# Eureka和Zookeeper的区别

## CAP理论

关于CAP理论，CAP理论就是说在分布式存储系统中，一个分布式系统不可能同时满足C(一致性)、A(可用性)和P(分区容错性)。

> 关系型数据库（MySQL、Oracle、SqlServer等）遵循的原则是：ACID（A:原子性、C:一致性、I:隔离性、D:持久性）
>
> 非关系型数据库（Redis、MongoDB等）遵循的原则是：CAP（C:强一致性、A:服务可用性、P:分区容错性）

由于分区容错性在是分布式系统中必须要保证的，因此我们只能在C和A之间进行权衡，而Zookeeper保证的是CP，Eureka保证的是AP。

## Zookeeper保证CP

&emsp;&emsp;当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接down掉不可用。也就是说，服务注册功能对可用性的要求要高于一致性。但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。

## Eureka保证AP

&emsp;&emsp;而Eureka却很好的保证了可用性，其在设计时就优先保证服务的可用性。Eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而Eureka的客户端在向某个Eureka注册或时如果发现连接失败，则会自动切换至其它节点，只要有一台Eureka还在，就能保证注册服务可用(保证可用性)，只不过查到的信息可能不是最新的(不保证强一致性)。除此之外，Eureka还有一种自我保护机制，如果在15分钟内超过85%的节点都没有正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况： 
1. Eureka不再从注册列表中移除因为长时间没收到心跳而应该过期的服务 
2. Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用) 
3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

因此， Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪。

# 总结

&emsp;&emsp;Eureka在作为单纯的服务注册中心来说比起Zookeeper要好上很多，因为注册服务需要优先保证的是可用性，而对于一致性并不是需要优先保证的原则，我们可以容忍短时间的不一致，但是不能容忍整个服务不可用的情况。