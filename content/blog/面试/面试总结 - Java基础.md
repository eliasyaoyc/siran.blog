---
title: "面试总结 - Java基础"
date: 2020-03-08T16:37:42+08:00
draft: true
banner: "/img/blog/banners/006tNc79ly1g1wrwnznblj31400u0x6p.jpg"
author: "Siran"
summary: ""
tags: ["Java基础"]
categories: ["面试"]
keywords: ["面试","Java基础"]
---

1. 讲一个集合框架整体框架

2. 分Collection和Map两大类全部讲一遍，每一个接口和对应实现类，他们类各自的特点，性质，基本参数，能讲多深讲多深

3. ArrayList和HashSet区别

4. 为什么Hashmap要在8的时候树化

hashmap线程安全的方式

hashtable和concurrenthashmap的各自特点，能讲多深讲多深

为什么hashtable被弃用了，cch1.7之前和1.8之后的区别

锁的分类

乐观锁、悲观锁、自旋锁、读写锁、排它锁、共享锁、分段锁等等各自特点，以及它们在java中具体的实现方式

Spring IOC的底层实现

XML+dom4j+工厂+单例

索引不适用的条件

索引列上有函数，不满足最左前缀，使用了不等号，使用了范围查询等等

索引的分类

B-Tree索引，Hash索引，全文索引，单值索引、唯一索引、复合索引、聚簇索引、非聚簇索引等等，以及它们各自的特点

executors创建的几种线程池，直接new ThreadPoolExecutor，7个参数

线程池拒绝策略分别使用在什么场景

Spring AOP的底层实现

动态代理，newProxyInstance，cglib，ASM

讲一下代理模式

动态代理，静态代理

你都了解什么设计模式，他们在JDK中如何体现的

工厂，责任链，观察者，建造，代理，单例，原型等等在JDK中对应的体现。。。

JAVA中的几种基本数据类型是什么，各自占用多少字节。

String类能被继承吗，为什么。

String，Stringbuffer，StringBuilder的区别。

ArrayList和LinkedList有什么区别。

讲讲类的实例化顺序，比如父类静态数据，构造函数，字段，子类静态数据，构造函数，字段，当new的时候，他们的执行顺序。

用过哪些Map类，都有什么区别，HashMap是线程安全的吗,并发下使用的Map是什么，他们内部原理分别是什么，比如存储方式，hashcode，扩容，默认容量等。

JAVA8的ConcurrentHashMap为什么放弃了分段锁，有什么问题吗，如果你来设计，你如何设计。

有没有有顺序的Map实现类，如果有，他们是怎么保证有序的。

抽象类和接口的区别，类可以继承多个类么，接口可以继承多个接口么,类可以实现多个接口么。

继承和聚合的区别在哪。

IO模型有哪些，讲讲你理解的nio ，他和bio，aio的区别是啥，谈谈reactor模型。

反射的原理，反射创建类实例的三种方式是什么。

反射中，Class.forName和ClassLoader区别 。

描述动态代理的几种实现方式，分别说出相应的优缺点。

动态代理与cglib实现的区别。

为什么CGlib方式可以对接口实现代理。

final的用途。

写出三种单例模式实现 。

如何在父类中为子类自动完成所有的hashcode和equals实现？这么做有何优劣。

请结合OO设计理念，谈谈访问修饰符public、private、protected、default在应用设计中的作用。

深拷贝和浅拷贝区别。

数组和链表数据结构描述，各自的时间复杂度。

error和exception的区别，CheckedException，RuntimeException的区别。

请列出5个运行时异常。

在自己的代码中，如果创建一个java.lang.String类，这个类是否可以被类加载器加载？为什么。

说一说你对java.lang.Object对象中hashCode和equals方法的理解。在什么场景下需要重新实现这两个方法。

在jdk1.5中，引入了泛型，泛型的存在是用来解决什么问题。

这样的a.hashcode() 有什么用，与a.equals(b)有什么关系。

有没有可能2个不相等的对象有相同的hashcode。

Java中的HashSet内部是如何工作的。

什么是序列化，怎么序列化，为什么序列化，反序列化会遇到什么问题，如何解决。

java8的新特性。JVM知识

讲讲JAVA的反射机制。

Linux系统下你关注过哪些内核参数，说说你知道的。

Linux下IO模型有几种，各自的含义是什么。

epoll和poll有什么区别。

平时用到哪些Linux命令。

用一行命令查看文件的最后五行。

用一行命令输出正在运行的java进程。

介绍下你理解的操作系统中线程切换过程。

进程和线程的区别。

top 命令之后有哪些内容，有什么作用。

线上CPU爆高，请问你如何找到问题所在。多线程

多线程的几种实现方式，什么是线程安全。

volatile的原理，作用，能代替锁么。

画一个线程的生命周期状态图。

sleep和wait的区别。

sleep和sleep(0)的区别。

Lock与Synchronized的区别 。

synchronized的原理是什么，一般用在什么地方(比如加在静态方法和非静态方法的区别，静态方法和非静态方法同时执行的时候会有影响吗)
，解释以下名词：重排序，自旋锁，偏向锁，轻量级锁，可重入锁，公平锁，非公平锁，乐观锁，悲观锁。

用过哪些原子类，他们的原理是什么。

JUC下研究过哪些并发工具，讲讲原理。

用过线程池吗，如果用过，请说明原理，并说说newCache和newFixed有什么区别，构造函数的各个参数的含义是什么，比如coreSize，maxsize等。

线程池的关闭方式有几种，各自的区别是什么。

假如有一个第三方接口，有很多个线程去调用获取数据，现在规定每秒钟最多有10个线程同时调用它，如何做到。

spring的controller是单例还是多例，怎么保证并发的安全。

用三个线程按顺序循环打印abc三个字母，比如abcabcabc。

ThreadLocal用过么，用途是什么，原理是什么，用的时候要注意什么。

如果让你实现一个并发安全的链表，你会怎么做。

有哪些无锁数据结构，他们实现的原理是什么。

讲讲java同步机制的wait和notify。

CAS机制是什么，如何解决ABA问题。

多线程如果线程挂住了怎么办。

countdowlatch和cyclicbarrier的内部原理和用法，以及相互之间的差别(比如countdownlatch的await方法和是怎么实现的)。

对AbstractQueuedSynchronizer了解多少，讲讲加锁和解锁的流程，独占锁和公平所加锁有什么不同。

使用synchronized修饰静态方法和非静态方法有什么区别。

简述ConcurrentLinkedQueue和LinkedBlockingQueue的用处和不同之处。

导致线程死锁的原因？怎么解除线程死锁。

非常多个线程（可能是不同机器），相互之间需要等待协调，才能完成某种工作，问怎么设计这种协调方案。

用过读写锁吗，原理是什么，一般在什么场景下用。

开启多个线程，如果保证顺序执行，有哪几种实现方式，或者如何保证多个线程都执行完再拿到结果。

延迟队列的实现方式，delayQueue和时间轮算法的异同。TCP与HTTP

http1.0和http1.1有什么区别。

TCP三次握手和四次挥手的流程，为什么断开连接要4次,如果握手只有两次，会出现什么。

TIME_WAIT和CLOSE_WAIT的区别。

说说你知道的几种HTTP响应码，比如200, 302, 404。

当你用浏览器打开一个链接（如：http://www.javastack.cn）的时候，计算机做了哪些工作步骤。

TCP/IP如何保证可靠性，说说TCP头的结构。

如何避免浏览器缓存。

如何理解HTTP协议的无状态性。

简述Http请求get和post的区别以及数据包格式。

HTTP有哪些method

简述HTTP请求的报文格式。

HTTP的长连接是什么意思。

HTTPS的加密方式是什么，讲讲整个加密解密流程。

Http和https的三次握手有什么区别。

什么是分块传送。

Session和cookie的区别。架构设计与分布式

用java自己实现一个LRU。

如果有人恶意创建非法连接，怎么解决。

如何设计建立和保持100w的长连接。

说说你平时用到的设计模式。

MVC模式，即常见的MVC框架。

Mybatis的底层实现原理。
