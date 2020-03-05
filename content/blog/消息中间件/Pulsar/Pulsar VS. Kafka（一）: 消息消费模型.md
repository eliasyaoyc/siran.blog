---
title: "Pulsar VS. Kafka（一）: 消息消费模型"
date: 2020-03-04T13:37:42+08:00
draft: false
banner: "/img/blog/pulsar/pulsar.svg"
author: "Siran"
summary: "Pulsar的特性包括消息的持久化存储，多租户，多机房互联互备，加密和安全性等。有比较强的健壮性，高可用性和可预测的延迟等。"
tags: ["消息中间件"]
categories: ["消息中间件"]
keywords: ["消息中间件","Pulsar"]
---

### 简述
>Pulsar是一个Pub/Sub 模式的消息中间件，但是现在Pulsar逐渐的发展成了一个`Event Streaming Platform`
>
>它与Kafka有很多类似的地方，但是又有些不同。Pulsar的特性包括消息的持久化存储，多租户，多机房互联互备，加密和安全性等。有比较强的健壮性，高可用性和可预测的延迟等。
>
>而在Kafka中`多租户`的概念是没有的，在消息的 `ack` 和 `message Retention` 设计上也有所不同以及`消费模式`等。
**** 
### Architecture Overview
>Pulsar 的架构如下图：

![](/img/blog/pulsar/architecture.png)
****
#### Broker
>Broker 用来接收与发送消息，生产方连接到 broker 去生产消息，消费方连接到 broker 去消费消息。
在Broker 中有两个重要的`组件`
* `HTTP 服务器`用来接收`Pulsar admin` 发来的请求来控制Pulsar如果熟悉kafka的话类比kafka中的admin 和处理Producer 与Consumer来寻找Topic的`lookup`请求
* `Dispatcher` 它是基于自定义二进制协议的异步TCP服务器用于所有数据传输的。lookup请求也可以通过TPC来发送

`note：` 与kafka中的broker的区别在于有无状态。
**** 
#### Zookeeper
>Pulsar使用Zookeeper来存储元数据，集群配置，协调，并且在`Producer` 和`Consumer`发送lookup请求来寻找topic，也是在zk中寻找。
**** 
#### BookKeeper
>跟kafka的不同在于Pulsar 把整个存储层抽象化了 broker本身不会存储数据，而是通过`BookKeeper`来存储数据，所以说只要不断增加`BookKeeper`集群中的实例那么可以无限制的存储。
**** 
#### Load Balancer
>多个`Pulsar instance` 有很多`租户(NameSpace)`，每个租户下又有很多topic，要根据一定`策略`把这些topic均匀的分布在每个`Pulsar instance`(一致性hash)，而处理这些分配的操作的则是由`Load Balancer`来完成。
>与kafka不同是，kafka默认是`轮训`分配，当然也可以自定义分配策略根据每个消息`key`，Pulsar也支持自定义分配
****
#### Service Discovery
>在`Pulsar Cluster`中，Producer 与 Consumer 接发消息都要指定一个topic，那么你并不知道这个topic经过`load balancer` 之后分配到了那个broker上。所以一开始接发消息的时候，随机给集群中的一台broker发送lookup请求，
>然后服务器接受到之后，去Zookeeper中查找并返回给客户端，这时客户端就会发起TCP长链接随后就是正常的接发消息了。这个过程就是`Service Discovery`
**** 
#### 消息消费模型
**队列模型** 
>队列模型主要是采用无序或者共享的方式来消费消息。通过队列模型，用户可以创建多个消费者从单个管道中接收消息；
>当一条消息从队列发送出来后，多个消费者中的只有一个（任何一个都有可能）接收和消费这条消息。消息系统的具体实现决定了最终哪个消费者实际接收到消息。
>
>队列模型通常与无状态应用程序一起结合使用。无状态应用程序不关心排序，但它们确实需要能够确认（ack）或删除单条消息，以及尽可能地扩展消费并行性的能力。
>典型的基于队列模型的消息系统包括RabbitMQ和RocketMQ。

**流式模型** 
>相比之下，流模型要求消息的消费严格排序或独占消息消费。对于一个管道，使用流式模型，始终只会有一个消费者使用和消费消息。
>消费者按照消息写入管道的确切顺序接收从管道发送的消息。
>
>流模型通常与有状态应用程序相关联。有状态的应用程序更加关注消息的顺序及其状态。消息的消费顺序决定了有状态应用程序的状态。消息的顺序将影响应用程序处理逻辑的正确性。
>典型的基于队列模型的消息系统就是Kafka。
****
#### Pulsar 的消息消费模型
Pulsar通过`Subscription`，抽象出了统一的: `producer-topic-subscription-consumer` 消费模型。Pulsar的消息模型既支持`队列模型`，也支持`流模型`。

在Pulsar的消息消费模型中，Topic是用于发送消息的通道。每一个Topic对应着BookKeeper中的一个分布式日志。发布者发布的每条消息只在Topic中存储一次；存储的过程中，BookKeeper会将消息复制存储在多个存储节点上；Topic中的每条消息，可以根据消费者的订阅需求，多次被使用，每个订阅对应一个消费者组（Consumer Group）。

主题（Topic）是消费消息的真实来源。尽管消息仅在主题（Topic）上存储一次，但是用户可以有不同的订阅方式来消费这些消息：
* 消费者被组合在一起以消费消息，每个消费组是一个订阅。
* 每个Topic可以有不同的消费组。
* 每组消费者都是对主题的一个订阅。
* 每组消费者可以拥有自己不同的消费方式： 独占（Exclusive），故障切换（Failover），共享（Share）和 key-Share。

**Exclusive Mode**
![](/img/blog/pulsar/pulsar-exclusive-subscriptions.png)
**Failover Mode**
![](/img/blog/pulsar/pulsar-failover-subscriptions.png)

**Share Mode**
![](/img/blog/pulsar/pulsar-shared-subscriptions.png)

**Key-Share Mode**
![](/img/blog/pulsar/pulsar-key-shared-subscriptions.png)
![](/img/blog/pulsar/pulsar-subscription-modes.png)
**** 
#### Pulsar 的消息确认（ACK）
![](/img/blog/pulsar/ack.jpg)

**** 
#### Pulsar 的消息保留（Retention）
![](/img/blog/pulsar/retention-expiry.png)

**** 
#### Pulsar VS. Kafka
**** 


****
本文来自Pulsar社区，如果对Pulsar感兴趣，可通过下列方式参与Pulsar社区：

- Pulsar Slack频道: 
  https://apache-pulsar.slack.com/
  
  可自行在这里注册：
  https://apache-pulsar.herokuapp.com/

- Pulsar邮件列表: http://pulsar.incubator.apache.org/contact



有关Apache Pulsar项目的常规信息，请访问官网：
http://pulsar.incubator.apache.org/
此外也可关注Twitter帐号@apache_pulsar。

