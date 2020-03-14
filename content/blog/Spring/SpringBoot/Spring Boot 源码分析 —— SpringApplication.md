---
title: "Spring Boot 源码分析 —— SpringApplication"
date: 2020-03-10T15:37:42+08:00
draft: false
banner: "/img/blog/spring/spring.svg"
author: "Siran"
summary: ""
tags: ["spring boot"]
categories: ["spring"]
keywords: ["spring","Jdk源码","基础"]
---

### 概述
```c
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// <1> 开启 自动配置
@SpringBootApplication 
public class TestApplication {
    public static void main(String[] args) {
        // <2> 调用 SpringApplication#run() 方法启动Spring Boot 应用
        SpringApplication.run(MVCApplication.class, args);
    }
}
```
这是我们在启动 Spring Boot 时，最常使用的启动类。

****
### Spring Boot 启动过程
在上面代码中的 `<2>` 调用`SpringApplication#run`方法来启动 Spring Boot应用，就从这分析整个Spring Boot的启动过程
```c
//org.springframework.boot.SpringApplication中的静态方法
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class[]{primarySource}, args);
    }
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return (new SpringApplication(primarySources)).run(args);
    }
```
可以看到最终创建一个 `SpringApplication` 对象。 
****

#### 构造方法 
```c
/**
 * 资源加载器
 */
private ResourceLoader resourceLoader;
/**
 * 主要的 Java Config 类的数组
 */
private Set<Class<?>> primarySources;
/**
 * Web 应用类型
 */
private WebApplicationType webApplicationType;

/**
 * ApplicationContextInitializer 数组
 */
private List<ApplicationContextInitializer<?>> initializers;
/**
 * ApplicationListener 数组
 */
private List<ApplicationListener<?>> listeners;

public SpringApplication(Class<?>... primarySources) {
        this((ResourceLoader)null, primarySources);
    }

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        ...
        //主要的 Java Config 类的数组。在文初提供的示例，就是 TestApplication 类。
        this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
        //调用 WebApplicationType#deduceFromClasspath() 方法，通过 classpath ，判断 Web 应用类型。 REACTIVE / SERVLET
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        //初始化 initializers 属性
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        //初始化 listeners 属性
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        this.mainApplicationClass = this.deduceMainApplicationClass();
    }
```
****
#### run 方法 
```c
public ConfigurableApplicationContext run(String... args) {
        // <1> 创建 StopWatch 对象，并启动。StopWatch 主要用于简单统计 run 启动过程的时长。
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        // <2> 配置 headless 属性
		configureHeadlessProperty();
        // 获得 SpringApplicationRunListener 的数组，并启动监听
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
            // <3> 创建  ApplicationArguments 对象
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // <4> 加载属性配置。执行完成后，所有的 environment 的属性都会加载进来，包括 application.properties 和外部的属性配置。
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
            // <5> 打印 Spring Banner
			Banner printedBanner = printBanner(environment);
            // <6> 创建 Spring 容器。
			context = createApplicationContext();
            // <7> 异常报告器
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
            // <8> 主要是调用所有初始化类的 initialize 方法
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            // <9> 初始化 Spring 容器。
			refreshContext(context);
            // <10> 执行 Spring 容器的初始化的后置逻辑。默认实现为空
			afterRefresh(context, applicationArguments);
            // <11> 停止 StopWatch 统计时长
			stopWatch.stop();
            // <12> 打印 Spring Boot 启动的时长日志。
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
            // <13> 通知 SpringApplicationRunListener 的数组，Spring 容器启动完成。
			listeners.started(context);
            // <14> 调用 ApplicationRunner 或者 CommandLineRunner 的运行方法。
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
            // <14.1> 如果发生异常，则进行处理，并抛出 IllegalStateException 异常
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
            // <15> 通知 SpringApplicationRunListener 的数组，Spring 容器运行中。
			listeners.running(context);
		}
		catch (Throwable ex) {
            // <15.1> 如果发生异常，则进行处理，并抛出 IllegalStateException 异常
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);



		}
		return context;
	}
```
****
#### prepareEnvironment
```c

```
****
#### configureIgnoreBeanInfo
```c

```
****
#### createApplicationContext
```c

```
****
#### prepareContext
```c

```
****
#### refreshContext
```c

```
****
#### afterRefresh
```c

```
****
#### callRunners
```c

```