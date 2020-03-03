---
title: "ReentrantLock 源码分析"
date: 2020-01-05T19:12:42+08:00
draft: false
banner: "/img/blog/banners/00704eQkgy1fs1hvk6nt7j30rs0kunjm.jpg"
author: "Siran"
summary: "通常使用锁就是 synchronized，经过 jdk 的一系列优化引入偏向锁、轻量级锁、重量级锁等概念，性能也是有很大的提高。"
tags: ["AQS"]
categories: ["AQS","Jdk源码"]
keywords: ["AQS","Jdk源码","基础"]
---
## 问题

## 简述
* ReentrantLock 与 AbstractQueuedSynchronizer 的关系
* ReentrantLock 和 Synchronized 的区别？
* ReentrantLock 实现原理
* ReentrantLock 公平锁和非公平锁的区别？

## 源码分析
> 通常使用锁就是 synchronized，经过 jdk 的一系列优化引入偏向锁、轻量级锁、重量级锁等概念，性能也是有很大的提高。
> 但是如果你想更加细化的控制锁或者说需要支持中断、公平性、有条件的加锁，那么 synchronized 是无法满足你的。
> 而 AbstractQueuedSynchronizer 则能满足你，这里简称AQS。AQS 提供一个 FIFO 队列，来实现锁以及其他功能。
> 它是一个抽象类，需要子类继承并实现所需要的方法来管理同步状态。例如：ReentrantLock，ReentrantReadWriteLock ，CountDownLatch，Semaphore等。

<img src="/images/AQS/AQS实现类.png" width="600" height="400">

> 在这么多实现类中，比较常用的也就是 ReentrantLock 和 CountDownLatch。本文主要分析 ReentrantLock
* {% post_link ReentrantReadWriteLock 源码分析 ReentrantReadWriteLock 源码分析 %}
* {% post_link Semaphore 源码分析 Semaphore 源码分析 %}
* {% post_link CountDownLatch 源码分析 CountDownLatch 源码分析 %}
* {% post_link Semaphore 源码分析 Semaphore 源码分析 %}

### AbstractQueuedSynchronizer 内部实现
> AQS 内部维护着一个FIFO队列，该队列就是用来实现线程的并发访问控制。假如锁已经被获取了，那么其他线程就无法获取锁，AQS 会把无法获取锁的线程封装成一个个Node并放入FIFO队列中。
> 假如获取锁的线程释放了锁，那么会去这个队列中唤醒线程来获取锁(这种情况是非公平锁，公平锁会顺序唤醒队列中的线程来获取锁但是公平锁的性能会下降)


Node 的主要属性(双向队列) 
```c
static final class Node {
    // 表示节点的状态，其中包含的状态有：
    // CANCELLED：值为1，表示当前节点被取消；
    // SIGNAL：值为-1，表示当前节点的后继节点将要或者已经被阻塞，在当前节点释放的时候需要unpark后继节点；
    // CONDITION：值为-2，表示当前节点在等待condition，即在condition队列中；
    // PROPAGATE：值为-3，表示releaseShared需要被传播给后续节点（仅在共享模式下使用）；
    // 0：无状态，表示当前节点在队列中等待获取锁
    int waitStatus;
    // 前继节点
    Node prev;
    // 后继节点
    Node next;
    // 存储condition队列中的后继节点
    Node nextWaiter;
    // 当前线程。
    Thread thread;
    // 该变量对不同的子类实现具有不同的意义，对ReentrantLock来说，它表示加锁的状态
    // 无锁时state=0，有锁时state>0；
    // 第一次加锁时，将state设置为1；
    // 由于ReentrantLock是可重入锁，所以持有锁的线程可以多次加锁，经过判断加锁线程就是当前持有锁的线程时（即exclusiveOwnerThread==Thread.currentThread()），即可加锁，每次加锁都会将state的值+1，state等于几，就代表当前持有锁的线程加了几次锁;
    // 解锁时每解一次锁就会将state减1，state减到0后，锁就被释放掉，这时其它线程可以加锁；
    // 当持有锁的线程释放锁以后，如果是等待队列获取到了加锁权限，则会在等待队列头部取出第一个线程去获取锁，获取锁的线程会被移出队列；
    private volatile int state;
}
```

### ReentrantLock
> 子类都需要继承 AbstractQueuedSynchronizer 来进行具体的实现。 在 ReentrantLock 中有三个内部类 Sync、NonfairSync、FairSync，来实现公平和非公平的获取锁

### 核心参数
```c
// 内部类 继承 AbstractQueuedSynchronizer
private final Sync sync;
```

### Sync
```c
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
        
        // 抽象方法由NonfairSync、FairSync具体的实现
        abstract void lock();

        // 非公平的获取锁实现，在NonfairSync中直接调用。此方法能获取获就返回true 不能就false区别于 #lock()方法
        final boolean nonfairTryAcquire(int acquires) {
            //<1> 获取当前线程
            final Thread current = Thread.currentThread();
            //<2> 获取state状态，这个字段的详细解释在上面的Node主要参数中已经解释过了
            int c = getState();
            //<3> 0表示当前没有线程获取锁，那么通过CAS来修改这个state状态，+1  并且设置线程为当前线程然后返回true
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //<4> 不为0 说明已经被其他线程获取了，判断是不是当前线程获取的(可重入的特性)
            else if (current == getExclusiveOwnerThread()) {
                // 校验
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                // 设置state值
                setState(nextc);
                // 返回true
                return true;
            }
            //<5> 无法获取锁，直接返回false。
            return false;
        }

        // 非公平的释放锁实现，在NonfairSync中直接调用
        protected final boolean tryRelease(int releases) {
            //<1> state - 1 锁数量-1
            int c = getState() - releases;
            //<2> 判断是否是同一个线程
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            //<3> 判断是否还有锁没被释放,由于重入的关系，不是每次释放锁c都等于0，直到最后一次释放锁时，才会把当前线程释放
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    }
```
### NonfairSync
> 内部类：非公平模式
```c
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        //<1> 先判断能否直接获取锁，cas修改 能的话直接把获取锁的线程设置为当前线程
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                <2> 调用AQS中的acquire
                acquire(1);
        }
        //直接调用Sync中的#nonfairTryAcquire方法
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

### FairSync
> 内部类：公平模式
```c
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //<1> 获取state
            int c = getState();
            //<2> state=0表示当前队列中没有线程被加锁
            if (c == 0) {
                /*
                 * 首先判断是否有前继结点，如果没有则当前队列中还没有其他线程；
                 * 设置状态为acquires，即lock方法中写死的1（这里为什么不直接setState？因为可能同时有多个线程同时在执行到此处，所以用CAS来执行）；
                 * 设置当前线程独占锁。
                 */
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
           /*
            * 如果state不为0，表示已经有线程独占锁了，这时还需要判断独占锁的线程是否是当前的线程，原因是由于ReentrantLock为可重入锁；
            * 如果独占锁的线程是当前线程，则将状态加1，并setState;
            * 这里为什么不用compareAndSetState？因为独占锁的线程已经是当前线程，不需要通过CAS来设置。
            */
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

### 构造函数
```c
// 默认实现是非公平模式
public ReentrantLock() {
        sync = new NonfairSync();
    }
// 传入true就是公平模式
public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

### lock 方法 独占锁获取
> 根据构造函数来具体调用是非公平模式的还是公平模式下的#lock方法。其差异就是在#tryAcquire()的不同。在上面NonfairSync、FairSync中已经分析了
```c
//获取锁
public void lock() {
        sync.lock();
    }
//最终是调用AQS中的#acquire()方法
public final void acquire(int arg) {
        //<1> 根据非公平模式的还是公平模式调用其#tryAcquire方法尝试获取锁 在上面已经分析过了。
        if (!tryAcquire(arg) &&
            //<2> 如果无法获取锁那么调用#addWaiter()方法封装成Node，并且调用#acquireQueued()添加到队列尾部自旋的获取锁。
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //<3> 如果在自旋过程中 中断标志位为true 则调用此方法设置线程中断
            selfInterrupt();
    }
```
### addWaiter 方法
> 该方法就是根据当前线程创建一个Node，然后添加到队列尾部。
```c
private Node addWaiter(Node mode) {
    //<1> 根据当前线程创建一个Node对象
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    //<2> 判断tail是否为空，如果为空表示队列是空的，直接enq
    if (pred != null) {
        node.prev = pred;
        //<3> 这里尝试CAS来设置队尾，如果成功则将当前节点设置为tail，否则enq
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
### enq 方法
> 如果#addWaiter方法无法把Node添加到队列尾部，那么就会调用此方法添加到队列中
```c
private Node enq(final Node node) {
    //<1> 死循环重复直到成功
    for (;;) {
        Node t = tail;
        //<2> 如果tail为null 则说明当前队列是空，则必须创建一个Node节点并进行初始化
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            //<3> 尝试CAS来设置队尾
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
### acquireQueued 方法
> 该方法会不断的调用#tryAcquire方法来获取锁，直到成功为止
```c
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 中断标志位
        boolean interrupted = false;
        for (;;) {
            //<1> 获取前继节点
            final Node p = node.predecessor();
            //<2> 如果前继节点是head，则尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 设置head为当前节点（head中不包含thread）
                setHead(node);
                // 清除之前的head，走到这里说明获取锁成功了，那么原来的head要从队列中剔除
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //<3> 如果p不是head或者获取锁失败，判断是否需要进行park
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            //<4> 如果在循环过程中出现了异常，则执行#cancelAcquire方法，用于将该节点标记为取消状态
            cancelAcquire(node);
    }
}
```
这里有三个问题:
1.什么条件下需要park？
2.为什么要判断中断状态？
3.死循环不会引起CPU使用率飙升？

### shouldParkAfterFailedAcquire 方法
> 在上面的#acquireQueued方法中的<2>中如果p不是head节点或者获取锁失败那么会进入此方法判断是否需要进行park
```c
//这里的两个参数一个是该节点的前节点，和该节点
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //<1> 获取前节点的waitStatus，在Node的主要属性中已经分析过了。
    int ws = pred.waitStatus;
    //<2> 如果前一个节点的状态是SIGNAL，则需要park； 
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    //<3> 如果ws > 0，表示已被取消，删除状态是已取消的节点；
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    //<4> 其他情况，设置前继节点的状态为SIGNAL。
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
> 也就是说只有在前节点的状态是SIGNAL时，需要park。

### parkAndCheckInterrupt 方法
> 在#acquireQueued方法<3>中除了调用#shouldParkAfterFailedAcquire方法来判断是否需要park还调用了#parkAndCheckInterrupt方法来判断是否中断
```c
private final boolean parkAndCheckInterrupt() {
    //<1> shouldParkAfterFailedAcquire 方法如果返回true 需要进行park，那么这里进行park
    LockSupport.park(this);
    //<2> 判断当前线程是否中断，并且复位
    return Thread.interrupted();
}
```
> 这里注意一下#interrupted方法，如果当前线程是中断状态，则第一次调用该方法获取的是true，第二次则是false 而#isInterrupted方法则只是返回线程的中断状态，不执行复位操作。

如果acquireQueued执行完毕，返回中断状态，回到acquire方法中，根据返回的中断状态判断是否需要执行Thread.currentThread().interrupt()。

为什么要多做这一步呢？先判断中断状态，然后复位，如果之前线程是中断状态，再进行中断？

这里就要介绍一下park方法了。park方法是Unsafe类中的方法，与之对应的是unpark方法。简单来说，当前线程如果执行了park方法，也就是阻塞了当前线程，反之，unpark就是唤醒一个线程。
park的具体分析可以参考这篇文章：

[!Java的LockSupport.park()实现分析](https://blog.csdn.net/hengyunabc/article/details/28126139)

park与wait的作用类型，但是对中断的处理并不相同。如果当前线程不是中断的状体，那么park/wait是一样的都会等待unpark/notify唤醒。但是如果一个线程已经是中断的状态了，wait会报错java.lang.IllegalMonitorStateException。
而park会直接返回。

所以，知道了这一点，就可以知道为什么要进行中断状态的复位了：
* 如果当前线程是非中断状态，则在执行park时被阻塞，这是返回中断状态是false；
* 如果当前线程是中断状态，则park方法不起作用，会立即返回，然后parkAndCheckInterrupt方法会获取中断的状态，也就是true，并复位；
* 再次执行循环的时候，由于在前一步已经把该线程的中断状态进行了复位，则再次调用park方法时会阻塞。

所以，这里判断线程中断的状态实际上是为了不让循环一直执行，要让当前线程进入阻塞的状态。
如果不这样判断，那么在#acquireQueued方法中前一个线程在获取锁之后执行了很耗时的操作，那么岂不是要一直执行该死循环？这样就造成了CPU使用率飙升，这是很严重的后果。

### cancelAcquire 方法
>在acquireQueued方法的finally语句块中，如果在循环的过程中出现了异常，则执行cancelAcquire方法，用于将该节点标记为取消状态。该方法代码如下：
```c
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;
    //<1> 设置该节点不再关联任何线程
    node.thread = null;
    // Skip cancelled predecessors
    //<2> 通过前继节点跳过取消状态的node
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    //<3> 获取过滤后的前继节点的后继节点
    Node predNext = pred.next;
    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    //<4> 设置状态为取消状态
    node.waitStatus = Node.CANCELLED;
    //<5> 这里出现三种情况：
    /* 
     * If we are the tail, remove ourselves.
     * 1.如果当前节点是tail：
     * 尝试更新tail节点，设置tail为pred；
     * 更新失败则返回，成功则设置tail的后继节点为null
     */
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        /* 
         * 2.如果当前节点不是head的后继节点：
         * 判断当前节点的前继节点的状态是否是SIGNAL，如果不是则尝试设置前继节点的状态为SIGNAL；
         * 上面两个条件如果有一个返回true，则再判断前继节点的thread是否不为空；
         * 若满足以上条件，则尝试设置当前节点的前继节点的后继节点为当前节点的后继节点，也就是相当于将当前节点从队列中删除
         */
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            // 3.如果是head的后继节点或者状态判断或设置失败，则唤醒当前节点的后继节点
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```
在#cancelAcquire方法中从队列中剔除节点的三种情况:
#### 当前节点是tail
这种情况很简单，因为tail是队列的最后一个节点，如果该节点需要取消，则直接把该节点的前继节点的next指向null，也就是把当前节点移除队列。出队的过程如下：
<img src="/images/AQS/1.png" width="600" height="400">

#### 当前节点不是head的后继节点，也不是tail，也就是当中节点

<img src="/images/AQS/2.png" width="600" height="400">

这里将node的前继节点的next指向了node的后继节点，真正执行的代码就是如下一行：
>compareAndSetNext(pred, predNext, next);

#### 当前节点是head的后继节点
<img src="/images/AQS/3.png" width="600" height="400">

这里直接unpark后继节点的线程，然后将next指向了自己。

这里可能会有疑问，既然要删除节点，为什么都没有对prev进行操作，而仅仅是修改了next？

要明确的一点是，这里修改指针的操作都是CAS操作，在AQS中所有以compareAndSet开头的方法都是尝试更新，并不保证成功，图中所示的都是执行成功的情况。

那么在执行cancelAcquire方法时，当前节点的前继节点有可能已经执行完并移除队列了（参见setHead方法），所以在这里只能用CAS来尝试更新，而就算是尝试更新，也只能更新next，不能更新prev，因为prev是不确定的，否则有可能会导致整个队列的不完整，例如把prev指向一个已经移除队列的node。

什么时候修改prev呢？其实prev是由其他线程来修改的。回去看下shouldParkAfterFailedAcquire方法，该方法有这样一段代码：
```c
do {
    node.prev = pred = pred.prev;
} while (pred.waitStatus > 0);
pred.next = node;
```
该段代码的作用就是通过prev遍历到第一个不是取消状态的node，并修改prev。

这里为什么可以更新prev？因为shouldParkAfterFailedAcquire方法是在获取锁失败的情况下才能执行，因此进入该方法时，说明已经有线程获得锁了，
并且在执行该方法时，当前节点之前的节点不会变化（因为只有当下一个节点获得锁的时候才会设置head），所以这里可以更新prev，而且不必用CAS来更新。

### unlock 方法  独占锁释放
```c
public void unlock() {
        sync.release(1);
    }
```
### release 方法
> unlock 释放锁和#lock获取锁一样凑通过调用AQS的方法来实现。这里调用的是#release方法
```c
public final boolean release(int arg) {
    //<1> 尝试释放锁
    if (tryRelease(arg)) {
        //<2> 释放成功后unpark后继节点的线程
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
### tryRelease 方法
```c
//和#tryAcquire一样，需要子类复写
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
//在reentrantLock的内部类sync中的实现，在分析sync的时候已经分析过了。这里就不重复分析了
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
### unparkSuccessor 方法
> 当前线程被释放之后，需要唤醒下一个节点的线程，通过unparkSuccessor方法来实现：
```c
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
主要功能就是要唤醒下一个线程，这里s == null || s.waitStatus > 0判断后继节点是否为空或者是否是取消状态，然后从队列尾部向前遍历找到最前面的一个waitStatus小于0的节点，至于为什么从尾部开始向前遍历，回想一下cancelAcquire方法的处理过程，cancelAcquire只是设置了next的变化，
没有设置prev的变化，在最后有这样一行代码：node.next = node，如果这时执行了unparkSuccessor方法，并且向后遍历的话，就成了死循环了，所以这时只有prev是稳定的。

### condition 条件队列
```c
public Condition newCondition() {
        return sync.newCondition();
    }
```
condition在这篇文章中进行分析:
* {% post_link Condition 源码分析 Condition 源码分析 %}

## 总结 
1. ReentrantLock 通过继承 AbstractQueuedSynchronizer 来具体实现同步
2. ReentrantLock 主要就是使用了标记位state(在ReentrantLock中表是获取锁的次数)，和 FIFO 队列来处理锁的状态、获取和方式。并且大量使用CAS来保证修改状态的安全
3. ReentrantLock 的非公平锁是抢占式的获取锁，而公平锁是顺序去获取。可以对比内部类FairSync和unFairSync中的#tryAcquire()方法