---
title: "面试总结 - Mysql"
date: 2020-03-08T16:37:42+08:00
draft: true
banner: "/img/blog/banners/006tNc79ly1g1wrwnznblj31400u0x6p.jpg"
author: "Siran"
summary: ""
tags: ["Mysql"]
categories: ["面试"]
keywords: ["面试","Mysql"]
---


数据库隔离级别有哪些，各自的含义是什么，MYSQL默认的隔离级别是是什么。

什么是幻读。

MYSQL有哪些存储引擎，各自优缺点。

高并发下，如何做到安全的修改同一行数据。

乐观锁和悲观锁是什么，INNODB的标准行级锁有哪2种，解释其含义。

SQL优化的一般步骤是什么，怎么看执行计划，如何理解其中各个字段的含义。

数据库会死锁吗，举一个死锁的例子，mysql怎么解决死锁。

MYsql的索引原理，索引的类型有哪些，如何创建合理的索引，索引如何优化。

聚集索引和非聚集索引的区别。

select for update 是什么含义，会锁表还是锁行或是其他。

为什么要用Btree实现，它是怎么分裂的，什么时候分裂，为什么是平衡的。

数据库的ACID是什么。

某个表有近千万数据，CRUD比较慢，如何优化。

Mysql怎么优化table scan的。

如何写sql能够有效的使用到复合索引。

mysql中in 和exists 区别。

数据库自增主键可能的问题。

MVCC的含义，如何实现的。

你做过的项目里遇到分库分表了吗，怎么做的，有用到中间件么，比如sharding jdbc等,他们的原理知道么。

MYSQL的主从延迟怎么解决。

MyBatis 编程步骤
#{} 和 ${} 的区别是什么？
当实体类中的属性名和表中的字段名不一样 ，怎么办？
XML 映射文件中，除了常见的 select | insert | update | delete标 签之外，还有哪些标签？
Mybatis 动态 SQL 是做什么的？都有哪些动态 SQL ？能简述一下动态 SQL 的执行原理吗？
最佳实践中，通常一个 XML 映射文件，都会写一个 Mapper 接口与之对应。请问，这个 Mapper 接口的工作原理是什么？Mapper 接口里的方法，参数不同时，方法能重载吗？
Mapper 接口绑定有几种实现方式,分别是怎么实现的?
Mybatis 的 XML Mapper文件中，不同的 XML 映射文件，id 是否可以重复？
如何获取自动生成的(主)键值?
Mybatis 执行批量插入，能返回数据库主键列表吗？
在 Mapper 中如何传递多个参数?
Mybatis 是否可以映射 Enum 枚举类？
Mybatis 都有哪些 Executor 执行器？它们之间的区别是什么？
MyBatis 如何执行批量插入?
介绍 MyBatis 的一级缓存和二级缓存的概念和实现原理？
Mybatis 是否支持延迟加载？如果支持，它的实现原理是什么？
Mybatis 能否执行一对一、一对多的关联查询吗？都有哪些实现方式，以及它们之间的区别。
简述 Mybatis 的插件运行原理？以及如何编写一个插件？
Mybatis 是如何进行分页的？分页插件的原理是什么？
MyBatis 与 Hibernate 有哪些不同？
JDBC 编程有哪些不足之处，MyBatis是如何解决这些问题的？
Mybatis 比 IBatis 比较大的几个改进是什么？
Mybatis 映射文件中，如果 A 标签通过 include 引用了B标签的内容，请问，B 标签能否定义在 A 标签的后面，还是说必须定义在A标签的前面？
简述 Mybatis 的 XML 映射文件和 Mybatis 内部数据结构之间的映射关系？