---
title: "Semaphore 源码分析"
date: 2018-06-05T19:12:42+08:00
draft: false
banner: "/img/blog/banners/006tNc79ly1g2bd7nfv72j31400u0x6s.jpg"
author: "Siran"
summary: "基于微服务的架构是未来的趋势，但是实现这种架构会面临许多困难。现代应用架构远比过去的架构复杂，因此实现微服务架构将会带来了一系列特殊的挑战，而服务网格可以帮我们解决很多问题。"
tags: ["AQS"]
categories: ["并发编程"]
keywords: ["AQS","Jdk源码","基础"]
---

### 问题
* Semaphore是什么？
* Semaphore具有哪些特性？
* Semaphore通常使用在什么场景中？
* Semaphore的许可次数是否可以动态增减？
* Semaphore如何实现限流？
****
### 简述
>Semaphore(信号量)，用于限制同一时间对共享资源的访问次数上，也就是常说的限流。在Semaphore中保存着permit(许可)，
>
>每次调用`#acquire`方法都会消费一个许可，当没有许可了就会堵塞住，每次调用`#release()`都将归还一个许可。
****
### 使用方法
**javadoc 中的例子**
```c
class Pool {
    private static final int MAX_AVAILABLE = 100;
    private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);
 
    public Object getItem() throws InterruptedException {
      available.acquire();
      return getNextAvailableItem();
    }
 
    public void putItem(Object x) {
      if (markAsUnused(x))
        available.release();
    }
 
    protected Object[] items = ... whatever kinds of items being managed
    protected boolean[] used = new boolean[MAX_AVAILABLE];
 
    protected synchronized Object getNextAvailableItem() {
      for (int i = 0; i < MAX_AVAILABLE; ++i) {
        if (!used[i]) {
           used[i] = true;
           return items[i];
        }
      }
      return null; // not reached
    }
 
    protected synchronized boolean markAsUnused(Object item) {
      for (int i = 0; i < MAX_AVAILABLE; ++i) {
        if (item == items[i]) {
           if (used[i]) {
             used[i] = false;
             return true;
           } else
             return false;
        }
      }
      return false;
    }
  }
```
**大致意思是：getNextAvailableItem方法只能有100个线程来使用，如果第101的线程要来使用，那么不好意思，它会被封装成node进去AQS队列中等待。
当#putItem方法中的#release方法来释放一个许可之后他会被从AQS中被唤醒。**
****
### 源码分析
**Semaphore中包含了一个实现了AQS的同步器Sync，以及它的两个子类FairSync和NonFairSync，这说明Semaphore也是区分公平模式和非公平模式的。**
#### 内部类Sync
```c
abstract static class Sync extends AbstractQueuedSynchronizer {
        //在Semaphore 中的state表示许可，也就是当前能执行的线程数量。
        Sync(int permits) {
            setState(permits);
        }
        //获取许可次数
        final int getPermits() {
            return getState();
        }
        //  非公平模式尝试获取许可
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                //看看还有几个许可
                int available = getState();
                int remaining = available - acquires;
                // 如果剩余许可小于0了则直接返回
                // 如果剩余许可不小于0，则尝试原子更新state的值，成功了返回剩余许可
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
        //  释放许可 
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                //1.查看当前还有几个许可
                int current = getState();
                //2.加上这次释放的许可
                int next = current + releases;
                //3.检测溢出
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                //4.如果原子更新state的值成功，就说明释放许可成功，则返回true
                if (compareAndSetState(current, next))
                    return true;
            }
        }
        //减少许可
        final void reducePermits(int reductions) {
            for (;;) {
                // 看看还有几个许可
                int current = getState();
                // 减去将要减少的许可
                int next = current - reductions;
                // 检查溢出
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                //原子更新state的值，成功了返回true
                if (compareAndSetState(current, next))
                    return;
            }
        }
        //销毁许可
        final int drainPermits() {
            for (;;) {
                // 看看还有几个许可
                int current = getState();
                // 如果为0 直接返回，不然cas修改为0
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }
```
****
#### 内部类NonfairSync
```c
//Semaphore.NonfairSync
static final class NonfairSync extends Sync {
        //构造方法，调用父类的构造方法
        NonfairSync(int permits) {
            super(permits);
        }
        //尝试获取许可，调用父类的nonfairTryAcquireShared()方法
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
```
****
#### 内部类FairSync
```c
//Semaphore.FairSync
static final class FairSync extends Sync {
        //构造方法，调用父类的构造方法
        FairSync(int permits) {
            super(permits);
        }
        //尝试获取许可
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                //公平模式需要检测是否前面有排队的  如果有排队的直接返回失败
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                //没有排队的再尝试更新state的值
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```
**公平模式下，先检测前面是否有排队的，如果有排队的则获取许可失败，进入队列排队，否则尝试原子更新state的值。**
****
#### 构造方法
```c
//构造方法，创建时要传入许可次数，默认使用非公平模式
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
//构造方法，需要传入许可次数，及是否公平模式
public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```
****
#### acquire 方法
**获取一个许可，默认使用的是可中断方式，如果尝试获取许可失败，会进入AQS的队列中排队。**
```c
public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

// 通过传入的permits参数可以一下获取多个许可
public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

//AbstractQueuedSynchronizer.acquireSharedInterruptibly
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        //1.判断线程是否中断，如果中断则复位抛出中断异常
        if (Thread.interrupted())
            throw new InterruptedException();
        //2. 尝试获取锁 如果获取不到则进入队列排队3
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

//AbstractQueuedSynchronizer.doAcquireSharedInterruptibly
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        //1.创建共享节点让入AQS中
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                // 前继节点
                final Node p = node.predecessor();
                //2.前继节点是否是head，是的话调用#tryAcquireShared尝试获取锁， 拿到锁就调用#setHeadAndPropagate方法顺序的唤醒下个节点。注意一次只唤醒一个节点
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                //3.判断是否要进行堵塞 避免死循环造成cpu飙升
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                //4.处理过程发送错误，取消
                cancelAcquire(node);
        }
    }
```
在`#acquire`方法中`#tryAcquireShared`方法尝试获取锁已经在上面的内部类讲过了，公平和非公平的区别在于是否能顺序的获取的锁

****
#### acquireUninterruptibly 方法
非中断的获取许可 对应 #acquire方法
```c
public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }
// 通过传入的permits参数可以一下获取多个许可
public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }
//AbstractQueuedSynchronizer.acquireShared
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
//AbstractQueuedSynchronizer.doAcquireShared  和上面的#doAcquireSharedInterruptibly方法类似区别在于此方法不会抛出中断异常
private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
****
#### release 方法
```c
//释放许可
public void release() {
        sync.releaseShared(1);
    }
//释放多个许可，可以通过此方法动态的增加许可的数量
public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }

//AbstractQueuedSynchronizer.releaseShared 
public final boolean releaseShared(int arg) {
        //1.死循环释放许可
        if (tryReleaseShared(arg)) {
            //2.释放成功，唤醒后继节点
            doReleaseShared();
            return true;
        }
        return false;
    }
//AbstractQueuedSynchronizer.tryReleaseShared  释放许可
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
//AbstractQueuedSynchronizer.tryReleaseShared  唤醒后继节点
private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
****
### 总结 
1. **Semaphore 也叫信号量，通常用于控制同一时刻对共享资源的访问上，也就是限流场景**
2. **Semaphore 初始化的时候需要指定许可的次数，许可的次数是存储在state中**
3. **可以动态减少n个许可**
4. **可以调用release(int permits)，来释放许可， 这样可以做到动态的增加许可**