---
title: "Pulsar VS. Kafka（二）: 以Segment为中心的架构"
date: 2020-03-04T13:38:42+08:00
draft: false
banner: "/img/blog/pulsar/pulsar.svg"
author: "Siran"
summary: ""
tags: ["消息中间件"]
categories: ["消息中间件"]
keywords: ["消息中间件","Pulsar"]
---
### Pulsar的分层架构
Pulsar和其他消息系统最根本的不同是采用分层架构。Pulsar集群由两层组成：无状态服务层，由一组接收和传递消息的Broker组成；以及一个有状态持久层，由一组名为bookies的BookKeeper存储节点组成，可持久化地存储消息。 

下图显示了 Pulsar的部署。
![](/img/blog/pulsar/640.webp)

****
### Broker层--无状态服务层
Broker集群在 Pulsar 中形成无状态服务层。服务层是"`无状态的`"，因为Broker并不会在本地存储任何消息。所有存储的工作都由`BookKeeper`来进行存储。每个主题分区（`Topic Partition`）由Pulsar分配给某个Broker，该Broker称为该主题分区的所有者。 Pulsar生产者和消费者连接到主题分区的所有者Broker，以向所有者代理发送消息并消费消息。

如果一个Broker失败，Pulsar会自动将其拥有的主题分区移动到群集中剩余的某一个可用Broker中。这里要说的一件事是：由于Broker是`无状态的`，当发生Topic的迁移时，Pulsar只是将`所有权`从一个Broker转移到另一个Broker，在这个过程中，`不会有任何数据复制发生`。

熟悉Kafka的都知道，kafka 是基于`本地日志的`，它的`Leader Broker` 会把本地日志复制给集群中的其他节点，那么如果有Broker宕机了，会进行Rebalance。在数据重新复制期间，分区通常不可用，直到数据重新复制完成。
               
****                                                     
### BookKeeper层--持久化存储层
BookKeeper是Pulsar的`持久化存储层`。Pulsar中的`每个主题分区`本质上都是存储在BookKeeper中的`分布式日志`。

每个分布式日志又被分为`Segment分段`。 每个Segment分段作为BookKeeper中的一个Ledger，均匀分布并存储在BookKeeper群集中的多个Bookie（`Apache BookKeeper的存储节点`）中。
Segment的创建时机包括以下几种：`基于配置的Segment大小`；`基于配置的滚动时间`；或者`当Segment的所有者被切换。`

通过`Segment分段`的方式，主题分区中的消息可以`均匀`和`平衡`地分布在群集中的所有Bookie中。 这意味着主题分区的大小不仅受一个节点容量的限制； 相反，它可以扩展到整个BookKeeper集群的总容量。

下面的图说明了一个分为x个Segment段的主题分区。 每个Segment段存储3个副本。 所有Segment都分布并存储在4个Bookie中。

![](/img/blog/pulsar/641.webp)
****

### Segment为中心的存储
`存储服务的分层的架构` 和 `以Segment为中心的存储` 是Pulsar（使用Apache BookKeeper）的两个关键设计理念。 这两个基础为Pulsar提供了许多重要的好处：
* 无限制的主题分区存储
* 即时扩展、无需数据迁移
  * 无缝Broker 故障恢复
  * 无缝集群扩展
  * 无缝的存储（Bookie）故障恢复
* 独立的可扩展性

#### 无限制的主题分区存储
由于在Pulsar中整个`存储层`都被抽象了出来，主题分区被分割成Segment并在`Apache BookKeeper`中以`分布式方式存储`，因此主题分区的容量不受任何单一节点容量的限制。 
相反，主题分区可以扩展到整个BookKeeper集群的总容量，只需添加Bookie节点即可扩展集群容量。 这是Apache Pulsar支持存储`无限大小的流数据`，
并能够以高效，分布式方式处理数据的关键。 使用Apache BookKeeper的分布式日志存储，对于`统一消息服务和存储`至关重要。

#### 即时扩展、无需数据迁移
由于`消息服务`和`消息存储`分为两层，因此将主题分区从一个Broker移动到另一个Broker几乎可以瞬时内完成，而`无需任何数据重新平衡`（将数据从一个节点重新复制到另一个节点）。 
只需要把这个主题分区的所有权转移到另一个Broker就可以，这一特性对于高可用的许多方面至关重要，例如集群扩展；对Broker和Bookie失败的快速应对。

* #### 无缝Broker 故障恢复
`下图说明了`Pulsar如何处理Broker失败的示例。 在例子中`Broker 2`因某种原因（例如停电）而断开。 Pulsar检测到`Broker 2`已关闭，并立即将`Topic1-Part2`的所有权从`Broker 2`转移到`Broker 3`。
在Pulsar中`数据存储`和`数据服务分离`，所以当代理3接管`Topic1-Part2`的所有权时，它`不需要复制Partiton的数据`。 如果有新数据到来，它立即附加并存储为`Topic1-Part2`中的`Segment x + 1`。
 Segment x + 1被分发并存储在Bookie1, 2和4上。因为它不需要重新复制数据，所以所有权转移立即发生而不会牺牲主题分区的可用性。
![](/img/blog/pulsar/642.webp)

* #### 无缝集群扩展
下图说明了Pulsar如何处理集群的`容量扩展`。 当Broker 2将消息写入`Topic1-Part2`的Segment X时，将Bookie X和Bookie Y添加到集群中。 Broker 2立即发现新加入的`Bookies X和Y`。
然后Broker将尝试将`Segment X + 1`和`X + 2`的消息存储到新添加的Bookie中。 新增加的Bookie立刻被使用起来，流量立即增加，而不会重新复制任何数据。
除了机架感知和区域感知策略之外，Apache BookKeeper还提供`资源感知`的`放置策略`，以确保流量在群集中的所有存储节点之间保持平衡。
![](/img/blog/pulsar/643.webp)

* #### 无缝的存储（Bookie）故障恢复
下图说明了Pulsar（通过Apache BookKeeper）如何处理bookie的磁盘故障。 这里有一个磁盘故障导致存储在bookie 2上的Segment 4被破坏。Apache BookKeeper后台会检测到这个错误并进行复制修复。

Apache BookKeeper中的副本修复是`Segment（甚至是Entry）`级别的`多对多快速修复`，这比重新复制整个主题分区要精细，只会`复制必须的数据`。 这意味着Apache BookKeeper可以从bookie 3和bookie 4读取Segment 4中的消息，并在bookie 1处修复Segment 4。所有的副本修复都在后台进行，对Broker和应用透明。
即使有Bookie节点出错的情况发生时，通过添加新的可用的Bookie来替换失败的Bookie，所有Broker都可以继续接受写入，而不会牺牲主题分区的可用性。
![](/img/blog/pulsar/644.webp)

#### 独立的可扩展性
由于消息服务层和持久存储层是分开的，因此Apache Pulsar可以独立地扩展`存储层`和`服务层`。这种独立的扩展，更具成本效益：

当需要支持更多的`消费者或生产者`时，您可以简单地添加更多的`Broker`。主题分区将立即在Brokers中做`平衡迁移`，一些主题分区的所有权立即转移到新的Broker。

当您需要`更多存储空间`来将消息保存更长时间时，只需添加更多`Bookie`。通过智能资源感知和数据放置，流量将自动切换到新的Bookie中。 Apache Pulsar中不会涉及到不必要的数据搬迁，不会将旧数据从现有存储节点重新复制到新存储节点。
****
### 和Kafka对比
Kafka和Pulsar都有类似的消息概念。 客户端通过主题与消息系统进行交互。 每个主题都可以分为多个分区。 然而，Pulsar和Kafka之间的根本区别在于Kafka是以`分区为存储中心`，而Pulsar是以`Segment为存储中心`。

下面的图显示了以`分区为中心`和以`Segment`为中心的系统之间的差异。
![](/img/blog/pulsar/645.webp)
在Kafka中，分区只能存储在`单个节点`上并`复制到其他节点`，其容量受最小节点容量的限制。这意味着容量扩展需要对分区`重新平衡`，这反过来又需要重新`复制整个分区`，以平衡新添加的代理的数据和流量。
重新传输数据非常昂贵且容易出错，并且会消耗网络带宽和I/O。维护人员在执行此操作时必须非常小心，以避免破坏生产系统。

Kafka中分区数据的重新拷贝不仅发生在以分区为中心的系统中的群集扩展上。许多其他事情也会触发数据重新拷贝，例如`副本故障，磁盘故障或计算机的故障`。在数据重新复制期间，分区通常不可用，直到数据重新复制完成。例如，如果将分区配置为存储为3个副本，这时，如果丢失了一个副本，则必须重新复制完整个分区后，分区才可以再次可用。

在用户遇到故障之前，通常会忽略这种缺陷，因为许多情况下，在短时间内仅是对内存中缓存数据的读取。当数据被保存到磁盘后，用户将越来越多地不可避免地遇到数据丢失，故障恢复的问题，特别是在需要将数据长时间保存的场合。

相反，在Pulsar中，同样是以`分区为逻辑单元`，但是以`Segment为物理存储单元`。分区随着时间的推移会进行分段，并在整个集群中`均衡分布`，旨在有效地迅速地扩展。
Pulsar是以Segment为中心的，因此在扩展容量时`不需要数据重新平衡和拷贝`，旧数据不会被重新复制，这要归功于在BookKeeper中使用可扩展的以Segment为中心的分布式日志存储系统。

通过利用分布式日志存储，Pulsar可以最大化Segment放置选项，`实现高写入和高读取可用性`。 例如，使用BookKeeper，副本设置等于2，只要任何2个Bookie启动，就可以对主题分区进行写入。 对于读取可用性，只要主题分区的副本集中有1个处于活动状态，用户就可以读取它，而不会出现任何不一致。
****
本文参考自Pulsar社区，如果对Pulsar感兴趣，可通过下列方式参与Pulsar社区：

- Pulsar Slack频道: 
  https://apache-pulsar.slack.com/
  
  可自行在这里注册：
  https://apache-pulsar.herokuapp.com/

- Pulsar邮件列表: http://pulsar.incubator.apache.org/contact



有关Apache Pulsar项目的常规信息，请访问官网：
http://pulsar.incubator.apache.org/
此外也可关注Twitter帐号@apache_pulsar。