---
title: "synchronized 的实现原理"
date: 2020-03-20T16:37:42+08:00
draft: false
banner: "/img/blog/banners/006tKfTcgy1ft5wypaul5j31ji15ob2b.jpg"
author: "Siran"
summary: "synchronized 关键字是 Java里面用来进行同步的。它编译后，会在同步块的前后分别生成 mointorenter 和 mointerexit 字节码指令，这两个字节码指令都需要一个引用类型的参数来指定要锁定和解锁的对象。"
tags: ["并发关键字"]
categories: ["并发编程"]
keywords: ["Java","基础"]
---

### 问题
1. synchronized是实现同步加锁的原理？
2. synchronized的优化？
3. synchronized的特性？
4. synchronized是否可重入？
5. synchronized是否是公平锁？
6. synchronized的优化？
7. synchronized锁膨胀过程
****
### 简介
synchronized 关键字是 Java里面用来进行同步的。它编译后，会在同步块的前后分别生成`mointorenter` 和 `mointerexit` 字节码指令，
这两个字节码指令都需要一个引用类型的参数来指定要锁定和解锁的对象。
****
### 使用方式
使用 `synchronized` 是需要一个`引用类型的参数`的，而这个引用类型的参数在Java中其实可以分成三大类：**类对象、实例对象、普通引用**，使用方式分别如下：
```c
public class SynchronizedTest {
    public static final Object lock = new Object();

    //锁定的是SynchronizedTest.class 对象
    public static synchronized void sync1() {
    }

    //锁定的是SynchronizedTest实例
    public synchronized void sync2() {

    }
    
    //锁定的是SynchronizedTest.class 对象
    public static void sync3() {
        synchronized (SynchronizedTest.class) {
        }
    }

    //锁的是指定对象lock
    public static void sync4() {
        synchronized (lock) {

        }
    }

    //锁定的是SynchronizedTest实例
    public void sync5() {
        synchronized (this) {

        }
    }
}
```
根据上面的代码中可以总结出来：**Java中的每一个对象都可以作为锁，具体表现以下三中：**

* 对于`普通同步`方法，锁是当前实例对象。
* 对于`静态同步`方法，锁是当前类的Class对象。
* 对于`同步方法`块，锁是Synchronized括号里配置的对象。

**另外，多个synchronized只有锁的是同一个对象，它们之间的代码才是同步的，这一点在使用synchronized的时候一定要注意。**
****
### 实现原理
在上面说到synchronized它编译后，会在同步块的前后分别生成`mointorenter` 和 `mointerexit` 字节码指令，这两个指令隐式的调用了 `lock` 和 `unlock` 指令。

* `lock`：锁定，**作用于主内存的变量，它把主内存中的变量标识为一条线程独占状态。**
* `unlock`：解锁，**作用于主内存的变量，它把锁定的变量释放出来，释放出来的变量才可以被其它线程锁定。**

**下面通过字节码来看一下Synchronized** 的实现：  
```c
public class synchronized Test {
    // 同步代码块
    public void doSth1(){
       synchronized (synchronizedTest.class){
           System.out.println("HelloWorld");
       }
    }
    // 同步方法
    public synchronized void doSth2(){
        System.out.println("HelloWorld");
    }
}
```
**使用javap对class文件进行反编译后结果**：
```c
public void doSth1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #2                  // class yichen/yao/SynchronizedTest
         2: dup
         3: astore_1
         4: monitorenter
         5: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: ldc           #4                  // String HelloWorld
        10: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        13: aload_1
        14: monitorexit
        15: goto          23
        18: astore_2
        19: aload_1
        20: monitorexit
        21: aload_2
        22: athrow
        23: return
      Exception table:
         from    to  target type
             5    15    18   any
            18    21    18   any
      LineNumberTable:
        line 40: 0
        line 41: 5
        line 42: 13
        line 43: 23
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      24     0  this   Lyichen/yao/SynchronizedTest;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 18
          locals = [ class yichen/yao/SynchronizedTest, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

 public synchronized void doSth2();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #4                  // String HelloWorld
         5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 47: 0
        line 48: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lyichen/yao/SynchronizedTest;
}
```
**从反编译后的结果中可以看到**：

* 对于`doSth1同步代码块`。JVM采用monitorenter、monitorexit两个指令来实现同步。`(第4、14、20行)`
* 对于`doSth2 同步方法`，JVM采用ACC_synchronized标记符来实现同步`(flags: ACC_PUBLIC, ACC_SYNCHRONIZED)`
****
#### 同步代码块
**在执行monitorenter指令的时候，根据JVM 规范有以下要求：**

![](/img/blog/并发编程/641.webp)

* 把执行`monitorenter`指令理解为加锁，执行`monitorexit`理解为释放锁。
* 每个对象维护着一个记录着被**锁次数的计数器**。
* 未被锁定的对象的该计数器为`0`，当一个线程获得锁（执行monitorenter）后，该计数器自增变为`1`，当同一个线程再次获得该对象的锁的时候，计数器再次`自增`。当同一个线程释放锁（执行monitorexit指令）的时候，计数器再`自减`。（可重入的性质）
* 当计数器为0的时候。锁将被释放，其他线程便可以获得锁。
****
#### 同步方法
**JVM采用ACC_synchronized标记符来实现同步。JVM规范介绍：**

![](/img/blog/并发编程/642.webp)

* 方法级的同步是`隐式`的。同步方法的常量池中会有一个`ACC_synchronized`标志。
* 当某个线程要访问某个方法的时候，会检查是否有ACC_synchronized，如果有设置，则需要先获得监视器锁（monitor），然后开始执行方法，方法执行之后再释放监视器锁。这时如果其他线程来请求执行方法，会因为无法获得监视器锁而被阻断住。
* 如果在方法执行过程中，发生了异常，并且方法内部并没有处理该异常，**那么在异常被抛到方法外面之前监视器锁会被自动释**放。对比 `doSth1同步代码块`，它在`第20行`，异常处理那也有一个`monitorexit`指令。用于在发生错误的时候可以退出同步块
****

### 保证原子性、可见性、有序性
Java 的内存模型主要就是用来解决`缓存一致性`的问题的，而缓存一致性主要包括**原子性、可见性、有序性**。

synchronized关键字底层是通过monitorenter和monitorexit实现的，而这两个指令又是通过lock和unlock来实现的。

**而lock和unlock在Java内存模型中是必须满足下面四条规则的**：

* `规则一`：一个变量同一时刻只允许一个线程对其进行lock操作，但lock操作可以被同一个线程执行多次，多次执行后，只有执行相同次数的unlock操作，变量才能被解锁。
* `规则二`：如果一个变量没有执行lock操作，将会情况工作内存中此变量的值，在执行引擎使用这个变量前，需要重新load 和 assign 操作初始化变量的值
* `规则三`：如果一个变量没有被lock锁定，则不允许使用unlock操作，也不允许unlock一个其他线程锁定的变量
* `规则四`：对一个变量执行unlock操作之前，必须先把此变量同步回主内存中，即执行store 和write操作。

**原子性：** 通过`规则一`，我们知道对于lock和unlock之间的代码，**同一时刻只允许一个线程操作**。所以保证了原子性。

**可见性：** 通过`规则一、二、四`，可以得出，每次lock和unlock时，都会从内存加载变量或把变量刷新回内存，而lock和unlock之间的变量是不会被其他线程修改的(`规则一：同一时刻只允许一个线程`)。所以保证了可见性

**有序性：** 通过`规则一、三`，我们知道所有对变量的加锁都要排队进行，且其它线程不允许解锁当前线程锁定的对象，所以，synchronized是具有有序性的。
****

### 锁优化
synchronized监视器锁在`互斥`同步上对性能的影响很大。

Java的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就需要操作系统的帮忙，这就要从`用户态转换到内核态`，状态转换需要花费很多的处理器时间。

所以频繁的通过Synchronized实现同步会严重影响到程序效率，这种锁机制也被称为`重量级锁`，为了减少重量级锁带来的性能开销，JDK对Synchronized进行了种种优化。
****
#### 自旋锁和适应自旋锁
**1）自旋锁**

* 当锁被占用时，当前想要获取锁的线程不会被立即挂起，而是做几个空循环，看持有锁的线程是否会很快释放锁。
* 在经过若干次循环后，如果得到锁，就顺利进入临界区；如果还不能获得锁，那就会将线程在操作系统层面挂起。

**2）自旋锁和阻塞最大的区别**

* 主要区别：是不是放弃处理器的执行时间。
* 阻塞放弃了CPU时间，进入了等待区，等待被唤醒。响应慢。自旋锁一直占用CPU时间，时刻检查共享资源是否可以被访问，所以响应速度更快。

**3）缺点**

* 如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好。但是如果持有锁的线程占用锁时间较长，等待锁的线程自旋一定次数后还是拿不到锁而被阻塞，那么自旋就白白浪费了CPU的资源。
* 所以自旋的次数直接决定了自旋锁的性能。**JDK自旋的默认次数为10次，可以通过参数-XX:PreBlockSpin来调整。**

**4）自适应自旋锁**

* 所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。
* 如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。
****
#### 锁消除
如果JVM检测到某段代码不可能存在`共享数据竞争`，JVM会对这段代码的同步锁进行`锁消除`。

在动态编译同步块的时候，JIT编译器可以借助一种被称为`逃逸分析（Escape Analysis）`的技术来判断同步块所使用的锁对象是否只能够被一个线程访问而没有被发布到`其他线程`。
如果同步块所使用的锁对象通过这种分析被证实只能够被一个线程访问，那么JIT编译器在编译这个同步块的时候就会**取消对这部分代码的同步**。
```c
public void vectorTest() {
    Vector<String> vector = new Vector<String>();
    for (int i = 0; i < 10; i++) {
        vector.add(i + "");
    }

    System.out.println(vector);
}
// Vector.add方法
public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
```
在运行这段代码时，JVM可以明显检测到变量vector没有逃逸出方法vectorTest()之外，**所以JVM可以大胆地将vector内部的加锁操作消除。**
****
#### 锁粗化
锁粗化就是将多个`连续`的加锁、`解锁`操作`连接在`一起，扩展成一个范围更大的锁。
这样做的原因是**如果在一段代码中连续的用同一个监视器锁反复的加锁解锁，甚至加锁操作出现在循环体中的时候，就会导致不必要的性能损耗，这种情况就需要锁粗化。**
```c
for(int i=0;i<100000;i++){
    synchronized(this){
        do();
}
```
如上面那串代码，会被粗化成：
```c
synchronized(this){
    for(int i=0;i<100000;i++){
        do();
}
```
****
#### Java对象头
对象在内存中存储的布局可以分为三块区域：`对象头（Header）`、`实例数据（Instance Data）`和`对齐填充（Padding）`。

![](/img/blog/并发编程/643.png)

普通对象的对象头包括两部分：`Mark Word`和`Class Metadata Address （类型指针）`，如果是数组对象还包括一个额外的`Array length`数组长度部分。
![](/img/blog/并发编程/企业微信截图_cf7d5ddb-575b-4ef5-b4a9-497ee73083c2.png)

* `Mark Word`：用于存储对象自身的运行时数据，如**哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，占用内存大小与虚拟机位长一致**。

* `Class Metadata Address`：类型指针指向对象的类元数据，虚拟机通过这个指针确定该对象是哪个类的实例。


**Mark Word**

对象头信息是与对象自身定义的数据无关的额外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态复用自己的存储空间。

**对mark word的设计方式上，非常像网络协议报文头：将mark word划分为多个比特位区间，并在不同的对象状态下赋予比特位不同的含义。**

下图描述了在32位虚拟机上，在对象不同状态时mark word各个比特位区间的含义。
![](/img/blog/并发编程/644.png)

****
#### 偏向锁、轻量级锁、重量级锁
从Java对象头的`Mark word`中可以看到，synchronized锁一共具有四种状态：**无锁、偏向锁、轻量级锁、重量级锁。**

偏向锁、轻量级锁、重量级锁三种形式，分别对应了锁只被`一个线程持有`、`不同线程交替持有锁`、`多线程竞争锁`三种情况。

**偏向锁**

* 目的：大多数情况下锁不仅不存在多线程竞争，而且总是由同一个线程多次获取，所以引入偏向锁让线程获得锁的代价更低。
* 偏向锁认为环境中不存在竞争情况，锁只被一个线程持有，一旦有不同的线程获取或竞争锁对象，偏向锁就升级为轻量级锁。
* 偏向锁在无多线程竞争的情况下可以减少不必须要的轻量级锁执行路径。

**轻量级锁**

* 目的：在大多数情况下同步块并不会出现竞争情况，大部分情况是不同线程交替持有锁，所以引入轻量级锁可以减少重量级锁对线程的阻塞带来的开销。
* 轻量级锁认为环境中线程几乎没有对锁对象的竞争，即使有竞争也只需要稍微等待（`自旋`）下就可以获取锁，但是自旋次数有限制，如果超过该次数，则会升级为重量级锁。

**重量级锁**

* 监视器锁Monitor
**** 
#### 锁的膨胀过程
synchronized锁膨胀过程就是**无锁 → 偏向锁 → 轻量级锁 → 重量级锁的一个过程**。这个过程是随着多线程对锁的`竞争越来越激烈，锁逐渐升级膨胀`的过程。
![](/img/blog/并发编程/201812081005.png)

**上图展示了synchronized 从无锁膨胀到重量级锁的一个过程：**

* **1)**  一个锁对象刚刚创建的时候，没有任何线程来访问它，此时线程状态为无锁状态。`Mark word 锁标记位-01 是否是偏向锁-0`

* **2)**  当`线程1`来访问这个对象锁时，它会偏向这个线程1。线程1检查**Mark word（锁标志位-01 是否偏向-0）为无锁状态**。此时，有线程访问锁了，`无锁升级为偏向锁`，**Mark word（锁标志位-01，是否偏向-1，线程ID-线程1的ID）**

* **3)**  当线程1 执行完同步块时，**持有偏向锁的线程不会`主动`释放偏向锁而是等待其他线程来竞争才会释放锁**。Mark word 不变(锁标志位-01，是否偏向-1，线程ID-线程1的ID)

* **4)**  当线程1再次获取这个`对象锁`时，检查Mark word（锁标志位-01，是否偏向-1，线程ID-线程1的ID），偏向锁且偏向线程1，可以`直接`执行同步代码。**这样偏向锁保证了总是同一个线程多次获取锁的情况下，每次只需要检查标志位就行，效率很高。**

* **5)**  当线程1执行完同步块之后，`线程2`获取这个对象锁 检查Mark word（锁标志位-01，是否偏向-1，线程ID-线程1的ID），偏向锁且偏向线程1。**有不同的线程获取锁对象，偏向锁升级为轻量级锁，并由线程2获取该锁。**

* **6)**  当线程1`正在执行`同步块时，也就是正持有偏向锁时，线程2获取来这个对象锁。**检查Mark word（锁标志位-01，是否偏向-1，线程ID-线程1的ID），偏向锁且偏向线程1。**
   * 线程1`撤销`偏向锁 :
     1. 等到全局安全点执行撤销偏向锁，**暂停持有偏向锁的线程1并检查程1的状态；**
     2. 如果线程1**不处于活动状态或者已经退出同步代码块**，则将对象锁设置为无锁状态，然后再升级为轻量级锁。由线程2获取轻量级锁。
     3. 如果线程1**还在执行同步代码块**，也就是线程1还需要这个对象锁，则偏向锁膨胀为轻量级锁。

   * 线程1`膨胀`为轻量级锁过程：
     1. 在升级为轻量级锁之前，持有偏向锁的线程（线程1）是`暂停`的
     2. 线程1栈帧中创建一个名为锁记录的空间（Lock Record）
     3. 锁对象头中的Mark Word拷贝到线程1的锁记录中
     4. Mark Word的锁标志位变为00，指向锁记录的指针指向`线程1`的锁记录地址，Mark word（锁标志位-00，其他位-线程1锁记录的指针）
     5. 当原持有偏向锁的线程（线程1）获取轻量级锁后，JVM唤醒线程1，线程1执行同步代码块
     
* **7)**  线程1持有轻量级锁，线程1执行完同步块代码之后，`一直没有线程来竞争对象锁`，正常`释放`轻量级锁。释放轻量级锁操作：**CAS操作将线程1的锁记录（Lock Record）中的Mark Word替换回锁对象头中。**
            
* **8)**  线程1持有轻量级锁，执行同步块代码过程中，`线程2来竞争对象锁`。 **Mark word（锁标志位-00，其他位-线程1锁记录的指针）**
     1. 线程2会先在栈帧中建立`锁记录`，存储锁对象目前的Mark Word的`拷贝`
     2. 线程2通过CAS操作尝试将锁对象的Mark Word的指针指向线程2的`Lock Record`，**如果成功，说明线程1刚刚释放锁，线程2竞争到锁，则执行同步代码块。**
     3. 因为线程1一直持有锁，大部分情况下CAS是会失败的。CAS失败之后，线程2尝试使用**自旋的方式来等待持有轻量级锁的线程释放锁。**
     4. 线程2不会一直自旋下去，如果自旋了`一定次数`后还是失败，线程2会被阻塞，等待释放锁后唤醒。此时轻量级锁就会膨胀为`重量级锁`。Mark word（锁标志位-10，其他位-重量级锁monitor的指针）
     5. 线程1执行完同步块代码之后，执行释放锁操作，**CAS 操作将线程1的锁记录（Lock Record）中的Mark Word 替换回锁对象对象头中，因为对象头中已经不是原来的轻量级锁的指针了，而是重量级锁的指针，所以CAS操作会失败。**
     6. 释放轻量级锁CAS操作替换失败之后，需要在释放锁的同时需要唤醒被挂起的线程2。线程2被唤醒，获取重量级锁monitor
****
锁膨胀的执行图：

![](/img/blog/并发编程/2bcc8161c52eb100d2c7c4c96c70d3c5823.jpg)

****
### 公平锁 VS 非公平锁
```c
public static void fair(String n)  {
        synchronized (SynchronizedTest.class){
            System.out.println(n);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        new Thread(()->fair("线程1")).start();
        new Thread(()->fair("线程2")).start();
        new Thread(()->fair("线程3")).start();
        new Thread(()->fair("线程4")).start();
    }
//打印结果
线程1
线程4
线程3
线程2
```
**由上面代码可以知道，只要有线程先获得锁，那就执行，所以synchronized是一个非公平锁**
**** 
### 总结
* **synchronized的实现原理：每一个Java对象都会关联一个Monitor，通过Monitor对线程的操作实现synchronized对象锁。**

* **monitorenter和monitorexit字节码指令更底层是使用Java内存模型的lock和unlock指令**

* **synchronized可以保证原子性、可见性、有序性。**

* **synchronized 是可重入锁、非公平锁**

* **自旋锁：当锁被占用时，当前想要获取锁的线程不会被立即挂起，而是做几个空循环，看持有锁的线程是否会很快释放锁。如果此时锁释放，当前线程就可以获得锁。`(CAS)`JDK自旋的默认次数为10次，可以通过参数-XX:PreBlockSpin来调整。**

* **自适应自旋锁：对比自旋锁，自旋次数不固定，JVM 由前一次在同一锁上的自旋时间及是否成功来判断，如果上次成功了，那么会增加自旋次数，如果失败，那么自旋次数减少甚至忽略自旋**

* **锁消除：如果JVM检测到某段代码不可能存在共享数据竞争，会对这段代码的同步锁进行锁消除。**

* **锁粗化：将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。**

* **synchronized有三种状态：偏向锁、轻量级锁、重量级锁，分别对应了锁只被一个线程持有、不同线程交替持有锁、多线程竞争锁三种情况。**

* **synchronized锁膨胀过程就是无锁 → 偏向锁 → 轻量级锁 → 重量级锁的一个过程。这个过程是随着多线程对锁的竞争越来越激烈，锁逐渐升级膨胀的过程。**

****
### 参考

《Java 并发编程实战》

《Java 并发编程的艺术》

[1. The Java® Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter)

[2. The Java® Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.10)

[synchronized原理](https://mp.weixin.qq.com/s?__biz=MzAxMjEwMzQ5MA==&mid=2448888833&idx=2&sn=1874f88dd96ee2102148f372ca5a544a&scene=21#wechat_redirect)

[synchronized锁优化](https://mp.weixin.qq.com/s?__biz=MzAxMjEwMzQ5MA==&mid=2448888881&idx=1&sn=75a4b57b369ff3c79ac372a6ba891101&scene=21#wechat_redirect)

[synchronized 的锁膨胀过程](http://cmsblogs.com/?p=5812)

[死磕 java同步系列之synchronized解析](https://mp.weixin.qq.com/s?__biz=Mzg2ODA0ODM0Nw==&mid=2247483919&idx=1&sn=e46cdc70c1f6bb08761a7dc2b4b5b2db&scene=21#wechat_redirect)