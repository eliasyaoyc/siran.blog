---
title: "Pulsar - Topic Discovery"
date: 2020-03-04T22:37:42+08:00
draft: false
banner: "/img/blog/pulsar/pulsar.svg"
author: "Siran"
summary: ""
tags: ["Pulsar"]
categories: ["Pulsar"]
keywords: ["消息中间件","Pulsar"]
---
#### Topic Assignment
Pulsar 作为`多租户`消息系统，具有`层级命名空间`，这个在之前我们也提到了「Tenant & Namespace」相关概念。除去前两层，第三层就是 topic。那么 Pulsar 如何把 topic 分配给 brokers 呢？

首先看图理解一下层次化结构。
![](/img/blog/pulsar/657.webp)
>一个 Pulsar instance 内部，有很多租户，就好比一个公司有多个不同的部门。往下细分又有不同的业务线（对应 namespace），业务线里可能又有不同的主题（对应 topic）。
>
>所有的 topic 在 Pulsar 集群里以树状结构连存，所以 topic 的分配是依照 `namespace` 层面进行划分的。

![](/img/blog/pulsar/658.webp)

>因为每个 namespace 下边有很多 topic，对于一个 namespace，它旗下的所有 topic 会组成一个`环`。在需要进行分配 topic 之前，会把这一组 topic 按照名字 hash 到 `namespace hash ring` 里。然后根据不同 topic 的 hash 值又会分为几个小组，也就是 `namespace bundle`，它的数量是在前期创建 namespace 时就可以指定确认。
>
>从 topic 到 bundle 映射完成后，Pulsar 接下来就会将 bundle 分配给 broker。也就是 topic 的分配不在于 topic 本身，而是依照 bundle 操作。处理过程依靠「`Load Manager`」进行，用来监控集群使用情况，并进行 bundle 分配。

![](/img/blog/pulsar/659.webp)
>选出一台 broker 作为 lead manager 主操作器，进行监控每台 broker 负载情况的同时，获取每个 namespace bundle 的数量值。通过一组计算后，根据负载情况来进行 topic 的分配。即所有都由 load manager 进行主动获取并分发相关。

![](/img/blog/pulsar/660.webp)
>以上就是 Pulsar 系统里，如何将 topic 分配到 broker 的过程。全程由 load manager 执行，按照 bundle 层面规则划分。
****
#### Topic lookup
在分配完成后，Pulsar 又是怎么正确找到哪台 broker 服务哪个 topic 的呢？首先提出一个新概念：`Topic owner`，换句话说就是` Namespace bundle owner`，可以依照上部分的层级结构进行对应梳理即可明白。

Owner 的相关信息以非持久化的状态储存在 ZooKeeper 里，任何一台 broker 都可以获取到 topic 到 owner 的映射关系。所以在客户端进行操作之前，会首先进行 topic lookup。具体过程如下：

Pulsar 客户端发起「`Topic lookup`」请求，请求会发给任意一台 broker。接收到请求后，broker 会开启 `lookup` 操作模式，根据 namespace 去检测出映射的 bundle，然后将此反馈发给 `ZooKeeper` 去查找对应，最后将请求结果返回给客户端。
![](/img/blog/pulsar/661.webp)
>这时客户端会发起 TCP 长链接，与 broker 2 进行直连。为了保证达到直连的效果，broker 2 提供的地址必须是 Pulsar 客户端可以直接链接的形式，这是非常重要的。

如果客户端不能直接访问代理地址，那应该如何处理呢？

在 Pulsar 里为了简化操作，在组件里添加了「`proxy`」配置，它是一个具有 topic 查询/路由 功能的 TCP 反向代理。
![](/img/blog/pulsar/662.webp)
也就是在 producer 并不是直接连接 broker，而是把请求发给 `proxy`，proxy 会根据 `topic-broker` 的分配记录进行安排。这样就会生成一个 TCP 长链接到 broker。

Proxy 的作用就是将这些涌入 TCP 的包通过探测来分辨请求类型，并基于此进行后续操作，即前后的连接作用。这样做的好处是不需要暴露 broker 地址，只需暴露 proxy 即可。

但这里也需要避开一个陷阱。很多人认为 proxy 起到了负载均衡的作用，它有一定的负载均衡连接数，`但真实的负载均衡仍然是在` broker 端进行的。所以这是两个层面的负载均衡操作，一定要分清。
![](/img/blog/pulsar/663.webp)
>有了 proxy 后，topic lookup 也可以更加充实一些。除了基本的操作反馈外，在客户端发起一个直连操作时，是可以将连接落实到任何一个 proxy 上。但是客户端的协议里，是带上了 topic owner，并传递给 proxy。

额外注意的是，加了 proxy 后的 topic lookup，需要保持 proxy 与 broker 在同一网络里。
****
**Topic lookup 的启动分为两种请求。**
1. Topic lookup 请求。可以给任意 broker 发送，客户端无需提前知道所有 broker 的地址。对于这种请求，可以在 broker 之前加上 DNS 或负载均衡器/配置多个 broker 地址去提供服务。

2. 知道 broker 后，需要建立 TCP 长链接的请求。额外注意的是 broker 的地址必须保证客户端与其可以直接连接。
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

