---
title: "面试总结 - Spring"
date: 2020-03-08T16:37:42+08:00
draft: false
banner: "/img/blog/banners/006tNc79ly1g1wrwnznblj31400u0x6p.jpg"
author: "Siran"
summary: ""
tags: ["Spring"]
categories: ["面试"]
keywords: ["Spring","面试"]
---

讲讲Spring加载流程。

Spring AOP的实现原理。

讲讲Spring事务的传播属性。

Spring如何管理事务的。

Spring怎么配置事务（具体说出一些关键的xml 元素）。

说说你对Spring的理解，非单例注入的原理？它的生命周期？循环注入的原理，aop的实现原理，说说aop中的几个术语，它们是怎么相互工作的。

Springmvc 中DispatcherServlet初始化过程。

springmvc用到的注解，作用是什么，原理。

springboot启动机制。


Spring 整体
什么是 Spring Framework？
Spring Framework 中有多少个模块，它们分别是什么？
使用 Spring 框架能带来哪些好处？
Spring 框架中都用到了哪些设计模式？
Spring IoC
什么是 Spring IoC 容器？
什么是依赖注入？
IoC 和 DI 有什么区别？
可以通过多少种方式完成依赖注入？
Spring 中有多少种 IoC 容器？
请介绍下常用的 BeanFactory 容器？
请介绍下常用的 ApplicationContext 容器？
列举一些 IoC 的一些好处？
简述 Spring IoC 的实现机制？
Spring 框架中有哪些不同类型的事件？
Spring Bean
什么是 Spring Bean ？
Spring 有哪些配置方式
Spring 支持几种 Bean Scope ？
Spring Bean 在容器的生命周期是什么样的？
什么是 Spring 的内部 bean？
什么是 Spring 装配？
解释什么叫延迟加载？
Spring 框架中的单例 Bean 是线程安全的么？
Spring Bean 怎么解决循环依赖的问题？
Spring 注解
什么是基于注解的容器配置？
如何在 Spring 中启动注解装配？
@Component, @Controller, @Repository, @Service 有何区别？
@Required 注解有什么用？
@Autowired 注解有什么用？
@Qualifier 注解有什么用？
Spring AOP
什么是 AOP ？
什么是 Aspect ？
什么是 JoinPoint ?
什么是 PointCut ？
关于 JoinPoint 和 PointCut 的区别
什么是 Advice ？
什么是 Target ？
AOP 有哪些实现方式？
Spring AOP and AspectJ AOP 有什么区别？
什么是编织（Weaving）？
Spring 如何使用 AOP 切面？
Spring Transaction
什么是事务？
事务的特性指的是？
列举 Spring 支持的事务管理类型？
Spring 事务如何和不同的数据持久层框架做集成？
为什么在 Spring 事务中不能切换数据源？
@Transactional 注解有哪些属性？如何使用？
什么是事务的隔离级别？分成哪些隔离级别？
什么是事务的传播级别？分成哪些传播级别？
什么是事务的超时属性？
什么是事务的只读属性？
什么是事务的回滚规则？
简单介绍 TransactionStatus ？
使用 Spring 事务有什么优点？
Spring Data Access
Spring 支持哪些 ORM 框架？
在 Spring 框架中如何更有效地使用 JDBC ？
Spring 数据数据访问层有哪些异常？
使用 Spring 访问 Hibernate 的方法有哪些？
Spring MVC
Spring MVC 框架有什么用？
介绍下 Spring MVC 的核心组件？
描述一下 DispatcherServlet 的工作流程？
@Controller 注解有什么用？
@RestController 和 @Controller 有什么区别？
@RequestMapping 注解有什么用？
@RequestMapping 和 @GetMapping 注解的不同之处在哪里？
返回 JSON 格式使用什么注解？
介绍一下 WebApplicationContext ？
Spring MVC 的异常处理？
Spring MVC 有什么优点？
Spring MVC 怎样设定重定向和转发 ？
Spring MVC 的 Controller 是不是单例？
Spring MVC 和 Struts2 的异同？
详细介绍下 Spring MVC 拦截器？
Spring MVC 的拦截器可以做哪些事情？
Spring MVC 的拦截器和 Filter 过滤器有什么差别？
REST
REST 代表着什么?
资源是什么?
什么是安全的 REST 操作?
什么是幂等操作? 为什么幂等操作如此重要?
REST 是可扩展的或说是协同的吗?
REST 用哪种 HTTP 方法呢?
删除的 HTTP 状态返回码是什么 ?
REST API 是无状态的吗?
REST安全吗? 你能做什么来保护它?
RestTemplate 的优势是什么?
HttpMessageConverter 在 Spring REST 中代表什么?
如何创建 HttpMessageConverter 的自定义实现来支持一种新的请求/响应？
@PathVariable 注解，在 Spring MVC 做了什么? 为什么 REST 在 Spring 中如此有用？
核心技术篇
Spring Boot 是什么？
Spring Boot 提供了哪些核心功能？
Spring Boot 有什么优缺点？
Spring Boot、Spring MVC 和 Spring 有什么区别？
Spring Boot 中的 Starter 是什么？
Spring Boot 常用的 Starter 有哪些？
创建一个 Spring Boot Project 的最简单的方法是什么？
如何统一引入 Spring Boot 版本？
运行 Spring Boot 有哪几种方式？
如何打包 Spring Boot 项目？
如果更改内嵌 Tomcat 的端口？
如何重新加载 Spring Boot 上的更改，而无需重新启动服务器？
Spring Boot 的配置文件有哪几种格式？
Spring Boot 默认配置文件是什么？
Spring Boot 如何定义多套不同环境配置？
Spring Boot 配置加载顺序？
Spring Boot 有哪些配置方式？
Spring Boot 的核心注解是哪个？
什么是 Spring Boot 自动配置？
Spring Boot 有哪几种读取配置的方式？
使用 Spring Boot 后，项目结构是怎么样的呢？
如何在 Spring Boot 启动的时候运行一些特殊的代码？
Spring Boot 2.X 有什么新特性？
整合篇
如何将内嵌服务器换成 Jetty ？
Spring Boot 中的监视器 Actuator 是什么？
如何集成 Spring Boot 和 Spring MVC ？
如何集成 Spring Boot 和 Spring Security ？
如何集成 Spring Boot 和 Spring Security OAuth2 ？
如何集成 Spring Boot 和 JPA ？
如何集成 Spring Boot 和 MyBatis ？
如何集成 Spring Boot 和 RabbitMQ ？
如何集成 Spring Boot 和 Kafka ？
如何集成 Spring Boot 和 RocketMQ ？
Spring Boot 支持哪些日志框架？