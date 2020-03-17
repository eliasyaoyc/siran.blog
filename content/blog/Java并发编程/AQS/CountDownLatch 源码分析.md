---
title: "CountDownLatch 源码分析"
date: 2020-02-05T19:12:42+08:00
draft: false
banner: "/img/blog/banners/006tKfTcgy1ftdx65tgduj30rs0kugov.jpg"
author: "Siran"
summary: "CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。"
tags: ["AQS"]
categories: ["并发编程"]
keywords: ["AQS","Jdk源码","基础"]
---

### 问题
* CountDownLatch是什么？
* CountDownLatch具有哪些特性？
* CountDownLatch通常运用在什么场景中？
* CountDownLatch的初始次数是否可以调整？
****
### 简述
>CountDownLatch这个类能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。
>
>CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

**执行过程如下图所示：**

![](/img/blog/AQS/4.png)
****
### CountDownLatch的使用
```c
public class CountDownTest {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3);
        System.out.println("in " + Thread.currentThread().getName() + "...");
        System.out.println("before latch.await()...");
        for (int i = 1; i <= 3; i++) {
            new Thread("T" + i) {
                @Override
                public void run() {
                    System.out.println("enter Thread " + getName() + "...");
                    System.out.println("execute countdown...");
                    latch.countDown();
                    System.out.println("exit Thread" + getName() + ".");
                }
            }.start();
        }
        latch.await();
        System.out.println("in " + Thread.currentThread().getName() + "...");
        System.out.println("after latch.await()...");
    }
}
```
**创建了一个初始值为3的CountDownLatch对象latch，然后创建了3个线程，每个线程执行时都会执行latch.countDown()使计数器的值减1，而主线程在执行到latch.await()时会等待直到计数器的值为0。输出的结果如下：**
```c
in main...
before latch.await()...
enter Thread T1...
enter Thread T2...
execute countdown...
execute countdown...
enter Thread T3...
execute countdown...
exit ThreadT2.
exit ThreadT1.
exit ThreadT3.
in main...
after latch.await()...

```
可以看到CountDownLatch 是主要通过`#countDown`方法和`#await`方法来实现此功能得，接下来会详细分析这两个方法。
****
### 源码分析(AQS共享模式的实现,ReentrantLock是独占模式)
#### 构造方法
```c
//传入一个参数count，CountDownLatch也使用了内部类Sync来实现，Sync继承自AQS：
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```
****
#### sync 内部类
```c
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
    Sync(int count) {
        setState(count);
    }
    int getCount() {
        return getState();
    }
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```
这里调用了AQS类中的setState方法来设置count，AQS的state属性在ReentrantLock文章已经提到，它是AQS中的状态标识，具体的含义由子类来定义，可见这里把`state定义为数量`。
****
#### await 方法
```c
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
直接调用了AQS类中的`#acquireSharedInterruptibly`方法

****
#### acquireSharedInterruptibly 方法
```c
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
    //<1> 如果线程被中断则抛出异常,会归位
    if (Thread.interrupted())
        throw new InterruptedException();
    //<2> 尝试获取共享锁，该方法在Sync类中实现
    if (tryAcquireShared(arg) < 0)
        //<3> 如果获取失败，需要根据当前线程创建一个mode为SHARE的的Node放入队列中并循环获取
        doAcquireSharedInterruptibly(arg);
}
```
这里的`tryAcquireShared`方法在Sync中被重写。
****
#### tryAcquireShared 方法
```c
protected int tryAcquireShared(int acquires) {
    //根据状态来判断，如果state等于0的时候，说明计数器为0了，返回1表示成功，否则返回-1表示失败。
    return (getState() == 0) ? 1 : -1;
}
```
****
#### doAcquireSharedInterruptibly 方法
```c
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    //<1> 创建一个共享模式的节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                //<2> 如果 p == head 表示是队列的第一个节点，尝试获取
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //<3> 设置当前节点为head，并向后面的节点传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            <4> 执行失败取消节点
            cancelAcquire(node);
    }
}
```
****
#### setHeadAndPropagate 方法
```c
private void setHeadAndPropagate(Node node, int propagate) {
    //<1> 首先先将之前的head记录一下，用于下面的判断
    Node h = head; // Record old head for check below
    //<2> 设置当前节点为头节
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    //<3> 最后在判断是否需要唤醒。这里的propagate值是根据tryAcquireShared方法的返回值传入的，所以对于CountDownLatch来说，如果获取成功，则应该是1。
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
>如果h.waitStatus >= 0，表示是初始状态或者是取消状态，那么当propagate <= 0时将不唤醒节点。
>
>获取node的下一个节点s，如果s == null || s.isShared()则释放节点并唤醒。为什么下一个节点为null的时候也需要唤醒操作呢？仔细理解一下这句话：
>
>The conservatism in both of these checks may cause unnecessary wake-ups, but only when there are multiple racing acquires/releases, so most need signals now or soon anyway.
>
>这种保守的检查方式可能会引起多次不必要的线程唤醒操作，但这些情况仅存在于多线程并发的acquires/releases操作，所以大多线程数需要立即或者很快地一个信号。这个信号就是执行unpark方法。因为LockSupport在unpark的时候，相当于给了一个信号，即使这时候没有线程在park状态，之后有线程执行park的时候也会读到这个信号就不会被挂起。
>
>在简单点说，就是线程在执行时，如果之前没有unpark操作，在执行park时会阻塞该线程；但如果在park之前执行过一次或多次unpark（unpark调用多次和一次是一样的，结果不会累加）这时执行park时并不会阻塞该线程。
>
>所以，如果在唤醒node的时候下一个节点刚好添加到队列中，就可能避免了一次阻塞的操作。
>
>所以这里的propagate表示传播，传播的过程就是只要成功的获取到共享锁就唤醒下一个节点。`
****
#### doReleaseShared 方法
```c
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //<1> 如果head的状态是SIGNAL，证明是等待一个信号，这时尝试将状态复位；
            // 如果复位成功，则唤醒下一节点，否则继续循环。
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            //<2> 如果状态是0，尝试设置状态为传播状态，表示节点向后传播；
            // 如果不成功则继续循环。
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //<3> 如果头节点有变化，则继续循环
        if (h == head)                   // loop if head changed
            break;
    }
}
```
什么时候状态会是SIGNAL呢？回顾一下`shouldParkAfterFailedAcquire`方法：
```c
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
当状态不为`CANCEL或者是SIGNAL`时，为了保险起见，这里把状态都设置成了SIGNAL，然后会再次循环进行判断是否需要阻塞。

回到doReleaseShared方法，这里为什么不直接把SIGNAL设置为PROPAGATE，而是先把SIGNAL设置为0，然后再设置为PROPAGATE呢？

原因在于unparkSuccessor方法，该方法会判断当前节点的状态是否小于0，如果小于0则将h的状态设置为0，如果在这里直接设置为PROPAGATE状态的话，则相当于多做了一次CAS操作。unparkSuccessor中的代码如下：
```c
int ws = node.waitStatus;
if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);

```
>其实这里只判断状态为SIGNAL和0还有另一个原因，那就是当前执行doReleaseShared循环时的状态只可能为SIGNAL和0，因为如果这时没有后继节点的话，当前节点状态没有被修改，是初始的0；
如果在执行setHead方法之前，这时刚好有后继节点被添加到队列中的话，因为这时后继节点判断p == head为false，所以会执行shouldParkAfterFailedAcquire方法，将当前节点的状态设置为SIGNAL。
当状态为0时设置状态为PROPAGATE成功，则判断h == head结果为true，表示当前节点是队列中的唯一一个节点，所以直接就返回了；如果为false，则说明已经有后继节点的线程设置了head，这时不返回继续循环，但刚才获取的h已经用不到了，等待着被回收。
****
#### countDown 方法
```c
public void countDown() {
    sync.releaseShared(1);
}
```
这里是调用了AQS中的`releaseShared`方法。
****
#### releaseShared 方法
```c
public final boolean releaseShared(int arg) {
    //<1> 尝试释放共享节点，如果成功则执行释放和唤醒操作
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```
这里调用的`tryReleaseShared`方法是在`CountDownLatch`中的Sync类重写的，而`doReleaseShare`d方法已在上文中介绍过了。
****
#### tryReleaseShared 方法
这里设置state的操作需要循环来设置以确保成功。
```c
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        //<1> 获取计数器数量
        int c = getState();
        //<2> 为0是返回false表示不需要释放
        if (c == 0)
            return false;
        //<3> 否则将计数器减1
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```
****
#### 超时功能的await方法
对应于上文中提到的`doAcquireSharedInterruptibly`方法，还有一个提供了超时控制的`doAcquireSharedNanos`方法，代码如下：
```c
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
	if (nanosTimeout <= 0L)
	    return false;
	final long deadline = System.nanoTime() + nanosTimeout;
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
	    for (;;) {
	        final Node p = node.predecessor();
	        if (p == head) {
	            int r = tryAcquireShared(arg);
	            if (r >= 0) {
	                setHeadAndPropagate(node, r);
	                p.next = null; // help GC
	                failed = false;
	                return true;
	            }
	        }
	        nanosTimeout = deadline - System.nanoTime();
	        if (nanosTimeout <= 0L)
	            return false;
	        if (shouldParkAfterFailedAcquire(p, node) &&
	            nanosTimeout > spinForTimeoutThreshold)
	            LockSupport.parkNanos(this, nanosTimeout);
	        if (Thread.interrupted())
	            throw new InterruptedException();
	    }
	} finally {
	    if (failed)
	        cancelAcquire(node);
	}
}
```
与doAcquireSharedInterruptibly方法新增了以下功能：
* 增加了一个deadline变量表示超时的截止时间，根据当前时间与传入的nanosTimeout计算得出；
* 每次循环判断是否已经超出截止时间，即deadline - System.nanoTime()是否大于0，大于0表示已经超时，返回false，小于0表示还未超时；
* 如果未超时通过调用shouldParkAfterFailedAcquire方法判断是否需要park，如果返回true则再判断nanosTimeout > spinForTimeoutThreshold，spinForTimeoutThreshold是自旋的最小阈值，这里被Doug Lea设置成了1000，表示1000纳秒，也就是说如果剩余的时间不足1000纳秒，则不需要park。
****
### 总结 
1. **CountDownLatch的AQS共享模式的实现：**

   `调用await`
* 共享锁获取失败（计数器还不为0），则将该线程封装为一个Node对象放入队列中，并阻塞该线程。
* 共享锁获取成功（计数器为0），则从第一个节点开始依次唤醒后继节点，实现共享状态的传播。

   `调用countDown`
* 如果计数器不为0，则不释放，继续阻塞，并把state的值减1；
* 如果计数器为0，则唤醒节点，解除线程的阻塞状态。
2. **CountDownLatch表示允许一个或多个线程等待其它线程的操作执行完毕后再执行后续的操作**
3. **独占模式和共享模式得区别：**

   `相同点`
* 锁的获取和释放的判断都是由子类来实现的。

   `不同点`
   
* 独占功能在获取节点之后并且还未释放时，其他的节点会一直阻塞，直到第一个节点被释放才会唤醒。
* 共享功能在获取节点之后会立即唤醒队列中的后继节点，每一个节点都会唤醒自己的后继节点，这就是共享状态的传播。

>根据以上的总结可以看出，AQS不关心state具体是什么，含义由子类去定义，子类则根据该变量来进行获取和释放的判断，AQS只是维护了该变量，
>
>并且实现了一系列用来判断资源是否可以访问的API，它提供了对线程的入队和出队的操作，它还负责处理线程对资源的访问方式，
>
>例如：什么时候可以对资源进行访问，什么时候阻塞线程，什么时候唤醒线程，线程被取消后如何处理等。而子类则用来实现资源是否可以被访问的判断。
