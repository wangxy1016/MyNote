# Euraka

## Euraka简介

Eureke是实现服务注册和服务发现的工具。（和ZooKeeper类似，两者之间的区别下面讲述）Spring Cloud集成了Eureka,开箱即用。Eureka也可以细分为Eureka Server 和Eureka Client。

## Eurek基本原理

服务启动后向Eureka注册，Eureka-Server会将注册信息向其他Eureka-server进行同步，当消费者（client）调用生产者（server）,则向服务注册中心获取服务提供者地址，然后会将服务提供者地址缓存在本地，下次再调用时，则直接从本地缓存中取，完成一次调用。

## Eureka自我保护机制

服务提供者在启动后，周期性（默认30s）向Eureka Server发送心跳，来证明当前服务是可用的。默认配置中，Eureka-server在默认90s没有得到客户端的心跳，则会注销改server实例。但在现实生活中，因为微服务跨进程调用，网络通信往往会面临着各种问题，比如微服务状态正常，但是因为网络分区故障时，Eureka Server注销服务实例则会让大部分微服务不可用，这很危险，因为服务明明没有问题。

为了解决该问题，Eureka有自我保护机制，在Eureka Server配置

```
eureka.server.enable-self-preservation=true //开启自我保护机制
```

当Eureka Server节点在短时间内发生故障，那么该节点会进入自我保护模式，不会注销任何实例。等到故障消失，自动退出自我保护模式。（宁可放过一个，不可错杀一千）

## 作为注册中心，Eureka和ZK的区别

在讲解区别前先了解一个理论：CAP

CAP理论，一个分布式系统不可能同时满足C（一致性）、A（可用性）、P（分区容错性）。分布式系统必须要保证分区容错性，只能在CP和AP之间进行选择

### ZK保证CP

当向注册中心查询服务列表时，可以容忍注册中心返回的是几分钟之前的注册信息，但是不能接受服务直接挂掉。也就是说，服务注册功能对可用性的要求要高于一致性。但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。

### Eureka保证AP

Eureka看明白了这一点，因此在设计时就优先保证可用性。Eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而Eureka的客户端在向某个Eureka注册或时如果发现连接失败，则会自动切换至其它节点，只要有一台Eureka还在，就能保证注册服务可用(保证可用性)，只不过查到的信息可能不是最新的(不保证强一致性)。除此之外，Eureka还有一种自我保护机制，如果在15分钟内超过85%的节点都没有正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况： 

1. Eureka不再从注册列表中移除因为长时间没收到心跳而应该过期的服务
2.  Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用
3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

因此， Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪



