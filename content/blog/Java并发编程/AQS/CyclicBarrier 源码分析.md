---
title: "CyclicBarrier 源码分析"
date: 2020-02-04T19:12:42+08:00
draft: false
banner: "/img/blog/banners/006tKfTcgy1ftd20xaj6jj31ji15oqv6.jpg"
author: "Siran"
summary: "CyclicBarrier(回声栅栏)根据Javadoc描述，它会阻塞一组线程直到这些线程同时达到某个条件才继续执行。它就像一个栅栏一样，当一组线程都到达了栅栏处才继续往下走。"
tags: ["AQS"]
categories: ["并发编程"]
keywords: ["AQS","Jdk源码","基础"]
---
### 问题
1. CyclicBarrier 与 CountDownLatch 的区别
2. CyclicBarrier具有什么特性？
****
### 简述
>CyclicBarrier(回声栅栏)根据Javadoc描述，它会阻塞一组线程直到这些线程同时达到某个条件才继续执行。它就像一个栅栏一样，当一组线程都到达了栅栏处才继续往下走。
CyclicBarrier 是可以被复用的。

****
### 使用方法
比如吃早饭，需要人全部到齐在开始吃。代码如下：
```c
//第一种 创建CyclicBarrier的时候只传入了等待的数量
public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);
        for (int i = 0; i < 3; i++) {
            new Thread(()->{
                try {
                    System.out.println(Thread.currentThread().getName()+"线程达到");
                    cyclicBarrier.await();
                } catch (Exception e) {
                    return;
                }
                System.out.println(Thread.currentThread().getName()+"吃早饭");
            }).start();
        }
    }
//第二种 创建CyclicBarrier的时候多传入了一个runnable
public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3,()->{
            System.out.println(Thread.currentThread().getName()+"吃早饭");
        });
        for (int i = 0; i < 3; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()+"线程达到");
                try {
                    cyclicBarrier.await();
                } catch (Exception e) {
                    return;
                }
            }).start();
        }
    }
```
**输出：**
```bash
//第一段代码的输出
Thread-0线程达到
Thread-2线程达到
Thread-1线程达到
Thread-1吃早饭
Thread-2吃早饭
Thread-0吃早饭

//第二段代码的输出
Thread-0线程达到
Thread-2线程达到
Thread-1线程达到
Thread-1吃早饭
```
**这代码很简单，第一段代码等三个线程全部到达的时候，在一起做下面的事情。
第二段代码，多传入了一个执行的command。当三个线程全部达到的时候，会触发command进行执行，随机一个线程执行。**
****
### 源码分析
#### 主要属性
```c
//锁
private final ReentrantLock lock = new ReentrantLock();
//条件锁  
private final Condition trip = lock.newCondition();
//需要等待的线程数量
private final int parties;
//当唤醒的时候执行的命令
private final Runnable barrierCommand;
//代
private Generation generation = new Generation();
//当前代还需要等待的线程数
private int count;
```
****
#### 内部类
```c
private static class Generation {
        boolean broken = false;
    }
```
**用于控制CyclicBarrier的循环使用
比如，上面示例中的三个线程完成后进入下一代，继续等待三个线程达到栅栏处再一起执行，而CountDownLatch则做不到这一点，CountDownLatch是一次性的，无法重置其次数。**
****
#### 构造器
```c
//构造方法需要传入一个parties变量，也就是需要等待的线程数。
public CyclicBarrier(int parties) {
        this(parties, null);
    }
//也可以传入执行的命令，唤醒时调用
public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```
****
#### await 方法
每个需要在栅栏处等待的线程都需要显式地调用`await()`方法等待其它线程的到来。
```c
public int await() throws InterruptedException, BrokenBarrierException {
        try {
            // 调用dowait方法，不需要超时
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        //1.加锁
        lock.lock();
        try {
            //当前代
            final Generation g = generation;
            //校验
            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            //2.count值-1,如果index = 0说明线程全部都到了，如果在构造器的时候传入了执行指令，那么变会执行
            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //3. 调用下一代方法
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            //这个循环只有非最后一个线程可以走到，因为如果是最后一个线程在上面就会return
            for (;;) {
                try {
                    //4.如果设置了超时时间，那么会调用condition的await()方法
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        //超时等待方法
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }
                //检查
                if (g.broken)
                    throw new BrokenBarrierException();

                //正常情况下这里是不等的，因为最后一个线程会调用#nextGeneration方法修改当前代，那么这里肯定就不一样了
                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
//此方法会调用condition 中的#signalAll方法把其队列中的等待线程全部唤醒
private void nextGeneration() {
        // signal completion of last generation
        // 调用condition的signalAll()将其队列中的等待者全部转移到AQS的队列中
        trip.signalAll();
        // set up next generation
        // 重置count
        count = parties;
        // 进入下一代
        generation = new Generation();
    }
private void breakBarrier() {
        // 调用condition的signalAll()将其队列中的等待者全部转移到AQS的队列中
        generation.broken = true;
        // 重置count
        count = parties;
        // 进入下一代
        trip.signalAll();
    }
```
**dowait()方法里的整个逻辑分成两个部分：**
1. 最后一个线程走上面的逻辑，当count减为0的时候，打破栅栏，它调用nextGeneration()方法通知条件队列中的等待线程转移到AQS的队列中等待被唤醒，并进入下一代。
2. 非最后一个线程走下面的for循环逻辑，这些线程会阻塞在condition的await()方法处，它们会加入到条件队列中，等待被通知，当它们唤醒的时候已经更新换“代”了，这时候返回。
****
### 总结 
1. **CyclicBarrier会使一组线程阻塞在await()处，当最后一个线程到达时唤醒（只是从条件队列转移到AQS队列中）前面的线程大家再继续往下走**
2. **CyclicBarrier不是直接使用AQS实现的一个同步器**
3. **CyclicBarrier基于ReentrantLock及其Condition实现整个同步逻辑**
4. **CyclicBarrier与CountDownLatch的异同？**
   * 两者都能实现阻塞一组线程等待被唤醒
   * CyclicBarrier是最后一个线程到达时自动唤醒
   * CountDownLatch是通过显式地调用countDown()实现的
   * CyclicBarrier是通过重入锁及其条件锁实现的，CountDownLatch是直接基于AQS实现的
   * CyclicBarrier具有“代”的概念，可以重复使用，CountDownLatch只能使用一次
   * CyclicBarrier只能实现多个线程到达栅栏处一起运行
   * CountDownLatch不仅可以实现多个线程等待一个线程条件成立，还能实现一个线程等待多个线程条件成立