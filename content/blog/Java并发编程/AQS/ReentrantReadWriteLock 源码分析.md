---
title: "ReentrantReadWriteLock 源码分析"
date: 2020-02-05T11:12:42+08:00
draft: false
banner: "/img/blog/banners/00704eQkgy1fs77zshhihj30rs0kunab.jpg"
author: "Siran"
summary: "ReentrantReadWriteLock 是什么?"
tags: ["AQS"]
categories: ["并发编程"]
keywords: ["AQS","Jdk源码","基础"]
---

### 问题
* ReentrantReadWriteLock 是什么
* ReentrantReadWriteLock 具有哪些特性？
* ReentrantReadWriteLock是怎么实现读写锁的？
* 如何使用ReentrantReadWriteLock实现高效安全的TreeMap？
****

### ReentrantReadWriteLock 使用
在`ReentrantReadWriteLock`类中`javadoc`中的例子，代码如下：
```c
class RWDictionary {
    private final Map<String, Data> m = new TreeMap<String, Data>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    // 读锁
    private final Lock r = rwl.readLock();
    // 写锁
    private final Lock w = rwl.writeLock();
 
    public Data get(String key) {
      r.lock();
      try { return m.get(key); }
      finally { r.unlock(); }
    }
    public String[] allKeys() {
      r.lock();
      try { return m.keySet().toArray(); }
      finally { r.unlock(); }
    }
    public Data put(String key, Data value) {
      w.lock();
      try { return m.put(key, value); }
      finally { w.unlock(); }
    }
    public void clear() {
      w.lock();
      try { m.clear(); }
      finally { w.unlock(); }
    }
  }
```
****
### 源码分析
#### 主要属性
```c
    // 内部类  读锁
    private final ReentrantReadWriteLock.ReadLock readerLock;
    // 内部类 写锁
    private final ReentrantReadWriteLock.WriteLock writerLock;
    // 内部类 继承 AbstractQueuedSynchronizer 
    final Sync sync;
```
****
#### 构造函数
和`ReentrantLock`一样也分公平锁和非公平锁，默认是非公平锁。
```c
public ReentrantReadWriteLock() {
        this(false);
    }
public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```
**在创建 ReentrantReadWriteLock 的时候会初始化两个内部类 ReadLock 和 WriteLock。
此外的另外三个内部类Sync、FairSync、NonfairSync 在ReentrantLock中已经分析过了详情可看下面这篇文章：**

****
### ReadLock 内部类
#### ReadLock.lock 方法
```c
//ReentrantReadWriteLock.ReadLock.lock()
public void lock() {
            sync.acquireShared(1);
        }
```
****
#### acquireShared 方法
`ReadLock.lock` 方法会调用 AQS中的`acquireShared`方法(共享模式)
```c
//AbstractQueuedSynchronizer.acquireShared()
public final void acquireShared(int arg) {
        //<1> 调用#tryAcquireShared方法尝试获取锁 如果能获取那就返回1，反之-1
        if (tryAcquireShared(arg) < 0)
            //<2> 如果无法获取锁，那么调用#doAcquireShared方法进行排队
            doAcquireShared(arg);
    }
```
****
#### tryAcquireShared 方法
尝试索取锁
```c
//ReentrantReadWriteLock.Sync.tryAcquireShared()
protected final int tryAcquireShared(int unused) {
            //当前线程
            Thread current = Thread.currentThread();
            //AQS的state字段，在ReentrantReadWriteLock中表示锁的数量  高16位存放独占锁的一些信息，低16位存放共享锁信息
            int c = getState();
            //1. 是否有独占锁获取，如果是的话 判断是不是当前线程。
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                // 锁获取失败 返回false
                return -1;
            //2.获取共享锁的数量
            int r = sharedCount(c);
            //3. 读操作不需要堵塞的话（是否是公平模式），判断共享锁的数量是否超过最大值65384，cas+1
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //3.1 如果共享锁的数量为0，那么firstReader设置为自己
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                //3.2 可重入的关系，如果firstReader是自己那么 holdCount+1
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                //3.3 如果firstReader 不是当前线程，那么从缓存中取出来 给该firstReader的holdCount+1
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                // 获取读锁成功，返回1
                return 1;
            }
            //4. 通过这个方法再去尝试获取读锁（如果之前其它线程获取了写锁，一样返回-1表示失败）
            return fullTryAcquireShared(current);
        }
```
****
#### doAcquireShared 方法
```c
//AbstractQueuedSynchronizer.doAcquireShared()
private void doAcquireShared(int arg) {
        //1. 封装一个shared 节点进入AQS的队列中
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //该节点的前一个节点
                final Node p = node.predecessor();
                //2.如果前一个节点是head节点的话(说明是第一个排队的节点)
                if (p == head) {
                    //3. 再次尝试获取锁
                    int r = tryAcquireShared(arg);
                    //4. r>0就是获取到了 ，上面说过
                    if (r >= 0) {
                        //5.头节点依次向后传播，即唤醒后续的读节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // 没获取到读锁，阻塞并等待被唤醒
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
**此方法与ReentrantLock中的方法`#acquireQueued`类似，以为的区别在于独占模式和共享模式
在此方法中如果当前节点获取到了锁则会调用`#setHeadAndPropagate`方法依次唤醒后续堵塞住的读节点**
****
#### setHeadAndPropagate 方法
```c
// AbstractQueuedSynchronizer.setHeadAndPropagate()
private void setHeadAndPropagate(Node node, int propagate) {
        //h为旧的头节点
        Node h = head; // Record old head for check below
        //1. 设置当前节点为头节点
        setHead(node);
        //2. 如果旧的头节点或新的头节点为空或者其等待状态小于0（表示状态为SIGNAL/PROPAGATE）
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            //3. 需要传播 取下一个节点
            Node s = node.next;
            //4. 如果下一个节点为null 或者是需要读锁的节点
            if (s == null || s.isShared())
                //5.唤醒
                doReleaseShared();
        }
    }
```
#### doReleaseShared 方法
唤醒节点，只会唤醒一个
```c
// AbstractQueuedSynchronizer.doReleaseShared()
private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //1. 如果头节点状态为SIGNAL，说明要唤醒下一个节点
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    //2. 唤醒下一个节点
                    unparkSuccessor(h);
                }
                //3. 把头节点的状态改为PROPAGATE成功才会跳到下面的if
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            //4. 如果唤醒后head没变，则跳出循环
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
**在梳理一下大致逻辑：**

1. 先调用#tryAcquireShared方法尝试获取锁，成功获取到直接返回
2. 没有获取到调用#doAcquireShared方法，会创建一个shared节点放入AQS中等待。
3. 在#doAcquireShared方法中，不断的尝试去获取锁
4. 如果头节点正好是当前节点的上一个节点，则设置该节点为头节点，并传播下去即唤醒下一个读节点(如果下一个节点是读节点的话)
4. 如果头节点不是当前节点的上一个节点或者4失败，堵塞当前线程等待唤醒，防止死循环
5. 唤醒之后继续走4的逻辑

****
#### ReadLock.unlock 方法
```c
//ReentrantReadWriteLock.ReadLock.unlock
public void unlock() {
            sync.releaseShared(1);
        }
//AbstractQueuedSynchronizer.releaseShared
public final boolean releaseShared(int arg) {
        //1. 尝试释放锁成功了，就唤醒下一个节点
        if (tryReleaseShared(arg)) {
            //2. 这个方法实际是唤醒下一个节点
            doReleaseShared();
            return true;
        }
        return false;
    }
//Sync.tryReleaseShared
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            //1.如果第一个读者（读线程）是当前线程
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                //获取锁的次数-1， 如果为0了就把firstReader 置为空
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                //2. 如果第一个读者不是当前线程。一样，把它重入的次数减1
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                //3. 共享锁获取的次数减1 如果减为0了说明完全释放了，才返回true
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
// AbstractQueuedSynchronizer.doReleaseShared()
private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //1.如果头节点状态为SIGNAL，说明要唤醒下一个节点
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    //2.唤醒下一个节点
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         //3.把头节点的状态改为PROPAGATE成功才会跳到下面的if
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            //4.如果唤醒后head没变，则跳出循环
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
**大致逻辑：**
1. 将当前线程重入的次数减1
2. 将共享锁总共被获取的次数减1
3. 如果共享锁获取的次数减为0了，说明共享锁完全释放了，那就唤醒下一个节点
****
### WriteLock 内部类
writeLock是一个独占模式的锁，可以参考ReentrantLock 一样的逻辑
#### WriteLock.lock 方法
```c
//ReentrantReadWriteLock.WriteLock.lock()
public void lock() {
            sync.acquire(1);
        }
//AbstractQueuedSynchronizer.acquire() 这边的逻辑和ReentrantLock 是一样的就不进行分析了
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
**大致过程这里在贴一下：**

1. 尝试获取锁；
2. 如果有读者占有着读锁，尝试获取写锁失败；
3. 如果有其它线程占有着写锁，尝试获取写锁失败；
4. 如果是当前线程占有着写锁，尝试获取写锁成功，state值加1；
5. 如果没有线程占有着锁（state==0），当前线程尝试更新state的值，成功了表示尝试获取锁成功，否则失败；
6. 尝试获取锁失败以后，进入队列排队，等待被唤醒；
7. 后续逻辑跟ReentrantLock是一致；

#### WriteLock.unlock 方法
也和ReentrantLoc的.unlock一样
```c
public void unlock() {
            sync.release(1);
        }
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
**写锁释放的过程大致为：**
1. 先尝试释放锁，即状态变量state的值减1；
2. 如果减为0了，说明完全释放了锁；
3. 完全释放了锁才唤醒下一个等待的节点；

### 总结 
1. **ReentrantReadWriteLock 采用读写锁的思想，提交并发的吞吐量。建议在多读少写的情况下使用，因为写锁是互斥的。**
2. **读锁使用的是共享锁，多个读锁可以一起获取锁，互相不会影响，即读读不互斥。**
3. **读写、写读和写写是会互斥的，前者占有着锁，后者需要进入AQS队列中排队。**
4. **多个连续的读线程是一个接着一个被唤醒的，而不是一次性唤醒所有读线程。**
5. **只有多个读锁都完全释放了才会唤醒下一个写线程。**
6. **只有写锁完全释放了才会唤醒下一个等待者，这个等待者有可能是读线程，也可能是写线程。**