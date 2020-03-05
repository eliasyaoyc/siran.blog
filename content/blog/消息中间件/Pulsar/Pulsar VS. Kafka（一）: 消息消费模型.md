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

![](/img/blog/pulsar/architecture.png)

**** 
#### Pulsar 的消息消费模型
**** 
#### Pulsar 的消息确认（ACK）
**** 
#### Pulsar 的消息保留（Retention）
**** 
#### Pulsar VS. Kafka
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

