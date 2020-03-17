---
title: "Pulsar VS. Kafka (一): 消息消费模型"
date: 2020-03-04T13:37:42+08:00
draft: false
banner: "/img/blog/pulsar/pulsar.svg"
author: "Siran"
summary: "Pulsar的特性包括消息的持久化存储，多租户，多机房互联互备，加密和安全性等。有比较强的健壮性，高可用性和可预测的延迟等。"
tags: ["Pulsar"]
categories: ["消息队列"]
keywords: ["消息中间件","Pulsar"]
---

### 简述
Pulsar是一个`Pub/Sub` 模式的消息中间件，但是现在Pulsar逐渐的发展成了一个`Event Streaming Platform`

它与Kafka有很多类似的地方，但是又有些不同。Pulsar的特性包括消息的持久化存储，多租户，多机房互联互备，加密和安全性等。有比较强的健壮性，高可用性和可预测的延迟等。

而在Kafka中`多租户`的概念是没有的，在消息的 `ack` 和 `message Retention` 设计上也有所不同以及`消费模式`等。
**** 
### Architecture Overview
Pulsar 的架构如下图：

![](/img/blog/pulsar/architecture.png)
****
#### Broker
Broker 用来接收与发送消息，生产方连接到 broker 去生产消息，消费方连接到 broker 去消费消息。
在Broker 中有两个重要的`组件`
* `HTTP 服务器`用来接收`Pulsar admin` 发来的请求来控制Pulsar如果熟悉kafka的话类比kafka中的admin 和处理Producer 与Consumer来寻找Topic的`lookup`请求
* `Dispatcher` 它是基于自定义二进制协议的异步TCP服务器用于所有数据传输的。lookup请求也可以通过TPC来发送

**`note：` 与kafka中的broker的区别在于Pulsar broker 是`无状态`的。**
**** 
#### Zookeeper
Pulsar使用`Zookeeper`来存储元数据，集群配置，协调，并且在`Producer` 和`Consumer`发送lookup请求来寻找topic，也是在zk中寻找。
**** 
#### BookKeeper
跟kafka的不同在于Pulsar 把整个存储层抽象化了 broker本身不会存储数据，而是通过`BookKeeper`来存储数据，所以说只要不断增加`BookKeeper`集群中的实例那么可以无限制的存储。
**** 
#### Load Balancer
多个`Pulsar instance` 有很多`租户(NameSpace)`，每个租户下又有很多topic，要根据一定`策略`把这些topic均匀的分布在每个`Pulsar instance`(一致性hash)，而处理这些分配的操作的则是由`Load Balancer`来完成。
与kafka不同是，kafka默认是`轮训`分配，当然也可以自定义分配策略根据每个消息`key`，Pulsar也支持自定义分配
****
#### Service Discovery
在`Pulsar Cluster`中，Producer 与 Consumer 接发消息都要指定一个topic，那么你并不知道这个topic经过`load balancer` 之后分配到了那个broker上。所以一开始接发消息的时候，随机给集群中的一台broker发送lookup请求，
然后服务器接受到之后，去Zookeeper中查找并返回给客户端，这时客户端就会发起TCP长链接随后就是正常的接发消息了。这个过程就是`Service Discovery`
**** 
#### 消息消费模型
**队列模型** 

队列模型主要是采用无序或者共享的方式来消费消息。通过队列模型，用户可以创建多个消费者从单个管道中接收消息；
当一条消息从队列发送出来后，多个消费者中的`只有一个`（任何一个都有可能）接收和消费这条消息。消息系统的具体实现决定了最终哪个消费者实际接收到消息。

队列模型通常与无状态应用程序一起结合使用。无状态应用程序不关心排序，但它们确实需要能够确认（ack）或删除单条消息，以及尽可能地扩展消费并行性的能力。
典型的基于队列模型的消息系统包括RabbitMQ和RocketMQ。

![](/img/blog/pulsar/queue.png)

`消息处理的瓶颈在于消费这个队列的消费者数量即可以添加消费者实例来增加整个消费的能力但是顺序性无法保证`

****
**流式模型** 

相比之下，流模型要求消息的消费严格排序或独占消息消费。对于一个管道，使用流式模型，`始终只会有一个`消费者使用和消费消息。
消费者按照消息写入管道的确切顺序接收从管道发送的消息。

流模型通常与有状态应用程序相关联。有状态的应用程序更加关注消息的顺序及其状态。消息的消费顺序决定了有状态应用程序的状态。消息的顺序将影响应用程序处理逻辑的正确性。
典型的基于队列模型的消息系统就是Kafka。

![](/img/blog/pulsar/stream.png)

`消息处理的瓶颈在于分区数量即可以添加分区数量来增加整个消费的能力单分区里的顺序可以保证`
****
#### Pulsar 的消息消费模型
Pulsar通过`Subscription`，抽象出了统一的: `producer-topic-subscription-consumer` 消费模型。Pulsar的消息模型既支持`队列模型`，也支持`流模型`。

在Pulsar的消息消费模型中，Topic是用于发送消息的通道。每一个Topic对应着BookKeeper中的一个分布式日志。发布者发布的每条消息只在Topic中存储一次；存储的过程中，BookKeeper会将消息复制存储在多个存储节点`Bookie`上；Topic中的每条消息，可以根据消费者的订阅需求，多次被使用，每个订阅(`Subscription`)对应一个消费者组（`Consumer Group`）。

主题（Topic）是消费消息的真实来源。尽管消息仅在主题（Topic）上存储一次，但是用户可以有不同的订阅方式来消费这些消息：
* 消费者被组合在一起以消费消息，每个消费组是一个订阅(`Subscription`)。
* 每个Topic可以有不同的消费组。
* 每组消费者都是对主题的一个订阅。
* 每组消费者可以拥有自己不同的消费方式： 独占（Exclusive），故障切换（Failover），共享（Share）和 Key-Share。
![](/img/blog/pulsar/pulsar-subscription-modes.png)
****
**Exclusive Mode（Stream流模型）**

在`Exclusive订阅`中，一个`消费组`（Subscription）只能有`一个`消费者来消费 Topic 中的消息。如图所示，如果另一个消费者A-1想要附加到这个消费组A(Subscription A)中，会被`拒绝`。这种模式就为 Pulsar 订阅模式中的`独占订阅（Exclusive）`。
![](/img/blog/pulsar/pulsar-exclusive-subscriptions.png)
****
**Failover Mode（Stream流模型）**

`Failover`订阅，多个消费者可以附加到同一订阅。 但是，一个订阅中的所有消费者，只会有`一个`消费者被选为该订阅的`主消费者`。 其他消费者将被指定为故障转移消费者。
当主消费者断开连接时，分区将被重新分配给其中一个故障转移消费者，而新分配的消费者将成为新的主消费者。 发生这种情况时，所有未确认（ack）的消息都将传递给`新的主消费者`。 这类似于Kafka中的`Consumer Partition Rebalance`。(`当有Consumer 加入或者退出消费组的时候会触发Rebalance`)

下图是`Failover`的示例。 消费者B-0和B-1通过订阅B订阅消费消息。B-0是主消费者并接收所有消息。 B-1是故障转移消费者，如果消费者B-0出现故障，它将接管消费。
![](/img/blog/pulsar/pulsar-failover-subscriptions.png)
****
**Share Mode（Queue队列模型）**

`Share`订阅，在同一个订阅背后，用户按照应用的需求挂载任意多的消费者。 订阅中的所有消息以循环分发形式发送给订阅背后的多个消费者，并且一个消息仅传递给一个消费者。
当消费者断开连接时，所有传递给它但是未被确认（ack）的消息将被重新分配和组织，以便发送给该订阅上剩余的剩余消费者。

下图是`Share`订阅的示例。 消费者C-1，C-2和C-3都在同一主题上消费消息。 每个消费者接收大约所有消息的1/3。
如果想提高消费的速度，用户不需要不增加分区数量(`Stream流模型需要增加分区来提交消费速度`)，只需要在同一个订阅中添加更多的消费者。
![](/img/blog/pulsar/pulsar-shared-subscriptions.png)
****
**Key-Share Mode（Stream流和Queue队列结合的模型）**

Key_Shared 订阅模式是 2.4.0 以后一个新订阅模式。类似于共享订阅，但又不是按照循环模式，是按照 key 进行分发，比如同一特征（奇数、偶数等）。总的来说是融合了 Failover 的有序性和 Shared 的消费扩展性、更均衡的一种订阅模式。
![](/img/blog/pulsar/pulsar-key-shared-subscriptions.png)
**** 

**订阅模式的选择**

`Exclusion`和`Failover`订阅，仅允许一个消费者来使用和消费，每个对主题的订阅。这两种模式都按主题分区顺序使用消息。它们最适用于需要严格消息顺序的流（Stream）用例。

`Share`订阅允许每个主题分区有多个消费者。同一订阅中的每个消费者仅接收主题分区的一部分消息。共享订阅最适用于不需要保证消息顺序的队列（Queue）的使用模式，并且可以按照需要任意扩展消费者的数量。

而`Key-Share`订阅可以按照`key`进行指定规则的分发，从而达到`Share`订阅无法满足的消息顺序性。

Pulsar中的订阅(`Subscription`)实际上与Kafka中的`Consumer Group`的概念类似。创建订阅的操作很轻量化，而且具有高度可扩展性，用户可以根据应用的需要创建任意数量的订阅。
对同一主题的不同订阅，也可以采用不同的订阅类型。比如用户可以在同一主题上可以提供一个包含3个消费者的`Failover`订阅，同时也提供一个包含20个消费者的`Share`订阅，并且可以在不改变分区数量的情况下，向`Share`订阅添加更多的消费者。

除了统一消息API之外，由于Pulsar主题分区实际上是存储在BookKeeper中，它还提供了一个读取API（Reader），类似于消费者API（但Reader没有游标管理），以便用户完全控制如何使用Topic中的消息。
****
#### Pulsar 的消息确认（ACK）
由于分布式系统的特性，当使用分布式消息系统时，可能会发生故障。比如在消费者从消息系统中的主题消费消息的过程中，消费消息的消费者和服务于主题分区的消息代理（Broker）都可能发生错误。消息确认（ACK）的目的就是保证当发生这样的故障后，消费者能够从上一次停止的地方恢复消费，保证既不会丢失消息，也不会重复处理已经确认（ACK）的消息。
在Kafka中，恢复点通常称为Offset，更新恢复点的过程称为消息确认或提交Offset。

在Pulsar中，每个订阅中都使用一个专门的数据结构--游标（Cursor）来跟踪订阅中的每条消息的确认（ACK）状态。每当消费者在主题分区上确认消息时，游标都会更新。更新游标可确保消费者不会再次收到消息。

Pulsar提供两种消息确认方法，`单条确认`（Individual Ack）和`累积确认`（Cumulative Ack）。通过累积确认，消费者只需要确认它收到的最后一条消息。主题分区中的所有消息（包括）提供消息ID将被标记为已确认，并且不会再次传递给消费者。累积确认与Apache Kafka中的Offset更新类似。

Pulsar可以支持消息的单条确认，也就是选择性确认。消费者可以单独确认一条消息。 被确认后的消息将不会被重新传递。

>下图说明了单条确认和累积确认的差异（灰色框中的消息被确认并且不会被重新传递）。在图的上半部分，它显示了累计确认的一个例子，M12之前的消息被标记为acked。在图的下半部分，它显示了单独进行acking的示例。仅确认消息M7和M12 - 在消费者失败的情况下，除了M7和M12之外，其他所有消息将被重新传送。

![](/img/blog/pulsar/ack.jpg)
独占订阅或故障切换订阅的消费者能够对消息进行单条确认和累积确认；共享订阅的消费者只允许对消息进行单条确认。单条确认消息的能力为处理消费者故障提供了更好的体验。对于某些应用来说，处理一条消息可能需要很长时间或者非常昂贵，防止重新传送已经确认的消息非常重要。

这个管理Ack的专门的数据结构--游标（Cursor），由Broker来管理，利用BookKeeper的Ledger提供存储。

Pulsar提供了灵活的消息消费订阅类型和消息确认方法，通过简单的统一的API，就可以支持各种消息和流的使用场景。
**** 
#### Pulsar 的消息保留（Retention）
在消息被确认后，Pulsar的Broker会更新对应的游标。当Topic里面中的一条消息，被所有的订阅都确认ack后，才能删除这条消息。Pulsar还允许通过设置保留时间，将消息保留更长时间，即使所有订阅已经确认消费了它们。

>下图说明了如何在有2个订阅的主题中保留消息。订阅A在M6和订阅B已经消耗了M10之前的所有消息之前已经消耗了所有消息。这意味着M6之前的所有消息（灰色框中）都可以安全删除。订阅A仍未使用M6和M9之间的消息，无法删除它们。如果主题配置了消息保留期，则消息M0到M5将在配置的时间段内保持不变，即使A和B已经确认消费了它们。

![](/img/blog/pulsar/retention-expiry.png)

在消息保留策略中，Pulsar还支持消息生存时间（TTL）。如果消息未在配置的TTL时间段内被任何消费者使用，则消息将自动标记为已确认。 消息保留期消息TTL之间的区别在于：消息保留期作用于标记为已确认并设置为已删除的消息，而TTL作用于未ack的消息。 
>上面的图例中说明了Pulsar中的TTL。 例如，如果订阅B没有活动消费者，则在配置的TTL时间段过后，消息M10将自动标记为已确认，即使没有消费者实际读取该消息。


**** 
#### Pulsar VS. Kafka

**模型概念**

Kafka： Producer - topic - `consumer group` - consumer；

Pulsar：Producer - topic - `subscription` - consumer。

**消费模式**

Kafka： 主要集中在流（Stream）模式，对单个partition是独占消费，没有共享（Queue）的消费模式，想要提交消费能力只能增加`partition数量`；

Pulsar：提供了统一的消息模型和API。流（Stream）模式 -- 独占和故障切换订阅方式；队列（Queue）模式 -- 共享订阅的方式；流(Stream)模式和队列(Queue)模式结合的 -- key-share订阅

**消息确认（Ack）**

Kafka： 使用偏移Offset；

Pulsar：使用专门的Cursor管理。累积确认和Kafka效果一样；提供单条或选择性确认。

**消息保留**

Kafka：根据设置的保留期来删除消息。有可能消息没被消费，过期后被删除。 不支持TTL。

Pulsar：消息只有被所有订阅消费后才会删除，不会丢失数据。也允许设置保留期，保留被消费的数据。支持TTL。

**对比总结：**

Pulsar将高性能的流（Kafka所追求的）和灵活的传统队列（RabbitMQ所追求的）结合到一个统一的消息模型和API中。 Pulsar使用统一的API为用户提供一个支持流和队列的系统，且具有同样的高性能。


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

