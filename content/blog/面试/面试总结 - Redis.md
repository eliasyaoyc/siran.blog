---
title: "面试总结 - Redis"
date: 2020-03-08T16:37:42+08:00
draft: true
banner: "/img/blog/banners/006tNc79ly1g1wrwnznblj31400u0x6p.jpg"
author: "Siran"
summary: ""
tags: ["Redis"]
categories: ["面试"]
keywords: ["面试","Redis"]
---

常见的缓存策略有哪些，如何做到缓存(比如redis)与DB里的数据一致性，你们项目中用到了什么缓存系统，如何设计的。

如何防止缓存击穿和雪崩。

缓存数据过期后的更新如何设计。

redis的list结构相关的操作。

Redis的数据结构都有哪些。

Redis的使用要注意什么，讲讲持久化方式，内存设置，集群的应用和优劣势，淘汰策略等。

redis2和redis3的区别，redis3内部通讯机制。

当前redis集群有哪些玩法，各自优缺点，场景。

Redis的并发竞争问题如何解决，了解Redis事务的CAS操作吗。

Redis的选举算法和流程是怎样的。

redis的持久化的机制，aof和rdb的区别。

redis的集群怎么同步的数据的。

知道哪些redis的优化操作。

Reids的主从复制机制原理。

Redis的线程模型是什么。

请思考一个方案，设计一个可以控制缓存总体大小的自动适应的本地缓存。

如何看待缓存的使用（本地缓存，集中式缓存），简述本地缓存和集中式缓存和优缺点。本地缓存在并发使用时的注意事项。搜索

如何用 Redis 统计独立用户访问量？