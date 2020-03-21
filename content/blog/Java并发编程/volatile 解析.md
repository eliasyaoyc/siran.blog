---
title: "volatile 解析"
date: 2020-03-20T16:37:42+08:00
draft: false
banner: "/img/blog/Java基础/1.jpg"
author: "Siran"
summary: "在多线程并发编程中synchronized 和 volatile 扮演着很重要的角色，volatile是轻量级的 synchronized,它能保证共享变量在多处理器下的可见性"
tags: ["并发关键字"]
categories: ["并发编程"]
keywords: ["Java","基础"]
---
### 问题
1. volatile 是如何保证可见性的？
2. volatile 是如何禁止重排序的？
3. volatile 的实现原理？
4. volatile 的缺陷？
5. volatile有哪些特性，可以用来做什么？
****
### 简述
在多线程并发编程中`synchronized` 和 `volatile` 扮演着很重要的角色，volatile是`轻量`级的 synchronized,它能保证共享变量在多处理器下的`可见性`。

`可见性`是指：**当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。**
****

### volatile 的内存语义
当给一个共享变量声明成 `volatile` 之后，那么这个变量将会具有 **可见性、有序性**。但是**不保证这个变量具有原子性**。
****
#### 语义：可见性
volatile保证了不同线程对volatile修饰的共享变量进行操作时的`可见性`。**（任意线程）总是能看到对这个volatile变量最后的写入。**

![](/img/blog/并发编程/1584716286872.jpg)

在上图中：v1变量没有被修饰成 volatile，如果线程A 首先对 v1进行读取存入线程A的本地内存中，
线程B对v1变量进行修改后，线程A将无法获取，因为在它的本地内存中存在，直接返回，`这就是内存不可见。`

**而加了 volatile 关键字后，它能保证以下情况：**
* 一个线程**修改**volatile变量的值时，**该变量的新值会立即刷新到主内存中，这个新值对其他线程来说是立即可见的。**
* 一个线程**读取**volatile变量的值时，**该变量在本地内存中缓存无效，需要到主内存中读取。**

**来看下面两个例子来说明volatile的语义**

例子1: 共享变量不添加volatile
```c
public class VolatileTest {
    private static int v1;
    public static void checkFinish(){
        while (v1 == 0 ){
            //do something
        }
        System.out.println("finished");
    }
    public static void finish(){
        v1 = 1;
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> checkFinish()).start();
        Thread.sleep(100);
        finish();
        System.out.println("main finish");
    }
}
//输出
main finish
```
****
例子2: 共享变量添加volatile
```c
public class VolatileTest {
    private volatile static int v1;
    public static void checkFinish(){
        while (v1 == 0 ){
            //do something
        }
        System.out.println("finished");
    }
    public static void finish(){
        v1 = 1;
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> checkFinish()).start();
        Thread.sleep(100);
        finish();
        System.out.println("main finish");
    }
}
//输出
finished
main finish
```
在上面的代码中，针对`finished`共享变量，使用volatile修饰时这个程序可以正常结束，不使用volatile修饰时这个程序**永远不会结束**。

因为不使用volatile修饰时，checkFinished()所在的线程每次都是读取的它自己`工作内存`中的变量的值，这个值一直为0，所以一直都不会跳出while循环。

使用volatile修饰时，checkFinished()所在的线程每次都是从主内存中加载最新的值，当finished被主线程修改为1的时候，它会立即感知到，进而会跳出while循环。
****

#### 语义：有序性(禁止重排序)
普通变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获得正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致，因为一个线程的方法执行过程中无法感知到这点，这就是“`线程内表现为串行的语义`”。

**volatile关键字能禁止这种行为，保证了程序会严格按照代码的先后顺序执行**，即保证了`有序性`。

比如，下面的代码：
```c
int i = 0；
int j = 1;
```
上面两句话没有依赖关系，JVM在执行的时候为了充分利用CPU的处理能力，可能会先执行 int j=1;这句，也就是重排序了，但是在线程内是无法感知的。

看似没有什么影响，但是如果是在多线程环境下呢？
```c
class Volatile2Test{
    private static String name;
    private static volatile boolean initialized = false;

    public static void main(String[] args) {
        new Thread(()->{
            name = "123";
            initialized = true;
        }).start();

        new Thread(()-> {
            if(!initialized)
                LockSupport.parkNanos(TimeUnit.MICROSECONDS,100);
            System.out.println(name);
        }).start();
    }
}
```
这个例子很简单，线程1负责赋值name，线程2检测initialized是否为true，打印出name。

在这个例子中，如果initialized不使用volatile来修饰，可能就会出现`重排序`，比如在初始化配置之前把initialized的值设置为了true，这样线程2读取到这个值为true了，就去使用配置了，这时候可能就会出现错误。

（此处这个例子只是用于说明重排序，实际运行时很难出现。）

所以，重排序是站在另一个线程的视角的，因为在本线程中，是无法感知到重排序的影响的。

**而volatile变量是禁止重排序的，它能保证程序实际运行是按代码顺序执行的。**
****

#### 不保证原子性
原子性是指**一个操作是不可中断的，要全部执行完成，要不就都不执行。**

**比如 int a = 0； a++**

a++ 不是一个原子操作，因为它要先`获取`a的值然后`在+1 `，在把值`赋值`给a

```c
public class VolatileTest {
    public volatile int a = 0;

    public void increase() {
        a++;
    }

    public static void main(String[] args) {
        final VolatileTest test = new VolatileTest();
        for (int i = 0; i < 10; i++) {
            new Thread() {
                public void run() {
                    for (int j = 0; j < 1000; j++)
                        test.increase();
                };
            }.start();
        }

        while (Thread.activeCount() > 1) {
            // 保证前面的线程都执行完
            Thread.yield();
        }
        System.out.println(test.a);
    }
}
```
每次运行得到的结果，都是小于10000，原因就如上面所说。**a++ 不是一个原子操作，因为它要先获取a的值然后在+1 ，在把值赋值给a**

那么要保证a++的原子性，就是保证这三个操作在一个线程没有执行完之前，不能被其他线程执行。

一个可能的执行时序图如下：
![](/img/blog/并发编程/640.png)

关键一步：**线程2在读取a的值时，线程1还没有完成a=1的赋值操作，导致线程2读取到当前a=0，所以线程2的计算结果也是a=1。**

问题在于没有保证a++操作的原子性。如果保证a++的原子性，线程1在执行完三个操作之前，线程2不能执行a++，那么就可以保证在线程2执行a++时，读取到a=1，从而得到正确的结果。

**解决：**
* synchronized或者ReentrantLock保证原子性
* CAS来实现原子性操作，AtomicInteger修饰变量a。
****
### volatile 实现原理(内存屏障)
JMM通过插入`内存屏障`指令来禁止特定类型的重排序。

java编译器在生成字节码时，在volatile变量操作前后的指令序列中插入内存屏障来禁止特定类型的重排序。

**volatile内存屏障插入策略**：
* 在每个volatile写操作的前面插入一个`StoreStore`屏障。
* 在每个volatile写操作的后面插入一个`StoreLoad`屏障。
* 在每个volatile读操作的后面插入一个`LoadLoad`屏障。
* 在每个volatile读操作的后面插入一个`LoadStore`屏障。

>`Store`：数据对其他处理器可见（即：刷新到内存中）
>
>`Load`：让缓存中的数据失效，重新从主内存加载数据

#### volatile保证可见性原理
volatile内存屏障插入策略中有一条，“**在每个volatile写操作的后面插入一个StoreLoad屏障**”。
 
StoreLoad屏障会生成一个`Lock前缀`的指令，Lock前缀的指令在多核处理器下会引发了两件事：
 
1. 将当前处理器缓存行的数据写回到系统内存。
2. 这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。
 
#### volatile内存可见的写-读过程：
1. volatile修饰的变量进行写操作。
2. 由于编译期间JMM插入一个StoreLoad内存屏障，JVM就会向处理器发送一条Lock前缀的指令。
3. Lock前缀的指令将该变量所在缓存行的数据写回到主内存中，并使其他处理器中缓存了该变量内存地址的数据失效。
4. 当其他线程读取volatile修饰的变量时，本地内存中的缓存失效，就会到到主内存中读取最新的数据。
****  
### 总结
* **volatile可以保证可见性和有序性，不能保证原子性。**
* **volatile是通过插入内存屏障禁止重排序来保证可见性和有序性的。**
* **volatile关键字的使用场景必须是场景本身就是原子的。**
****
### 参考
《Java 并发编程的艺术》

[深入理解volatile](https://mp.weixin.qq.com/s?__biz=MzAxMjEwMzQ5MA==&mid=2448888521&idx=1&sn=c880331ef37b5f111f553ad0b4064bab&scene=21#wechat_redirect)

[死磕 java同步系列之volatile解析](https://mp.weixin.qq.com/s?__biz=Mzg2ODA0ODM0Nw==&mid=2247483914&idx=1&sn=46e6d95df8dc445da6a8f774dc128451&scene=21#wechat_redirect)