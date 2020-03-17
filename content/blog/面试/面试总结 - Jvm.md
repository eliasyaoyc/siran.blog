---
title: "面试总结 - Jvm"
date: 2020-03-08T16:37:42+08:00
draft: true
banner: "/img/blog/banners/006tNc79ly1g1wrwnznblj31400u0x6p.jpg"
author: "Siran"
summary: ""
tags: ["JVM"]
categories: ["面试"]
keywords: ["面试","JVM"]
---

讲一下JVM堆内存管理

栈上分配->TLAB->新生代、老年代->可达性分析->GC算法->所有垃圾回收器及其优缺点和特点

那到底多大的对象会被直接扔到老年代

G1两个region不是连续的，而且之间还有可达的引用，我现在要回收其中一个，另一个会被怎么处理

听说过CMS的并发预处理和并发可中断预处理吗

什么情况下会发生栈内存溢出。

JVM的内存结构，Eden和Survivor比例。

JVM内存为什么要分成新生代，老年代，持久代。新生代中为什么要分为Eden和Survivor。

JVM中一次完整的GC流程是怎样的，对象如何晋升到老年代，说说你知道的几种主要的JVM参数。

你知道哪几种垃圾收集器，各自的优缺点，重点讲下cms和G1，包括原理，流程，优缺点。

垃圾回收算法的实现原理。

当出现了内存溢出，你怎么排错。

JVM内存模型的相关知识了解多少，比如重排序，内存屏障，happen-before，主内存，工作内存等。

简单说说你了解的类加载器，可以打破双亲委派么，怎么打破。

你们线上应用的JVM参数有哪些。

g1和cms区别,吞吐量优先和响应优先的垃圾收集器选择。

怎么打出线程栈信息。

请解释如下jvm参数的含义：
-server -Xms512m -Xmx512m -Xss1024K
-XX:PermSize=256m -XX:MaxPermSize=512m -
XX:MaxTenuringThreshold=20XX:CMSInitiatingOccupancyFraction=80 -
XX:+UseCMSInitiatingOccupancyOnly

线上系统突然变得异常缓慢，你如何查找问题。
