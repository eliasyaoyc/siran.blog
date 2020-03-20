---
title: "ScheduledThreadPoolExecutor 源码分析"
date: 2020-01-05T12:12:42+08:00
draft: false
banner: "/img/blog/banners/006tNc79ly1g1wrp8d29yj31400u0kjm.jpg"
author: "Siran"
summary: "ScheduledThreadPoolExecutor 定义了一个延迟队列 DelayedWorkQueue，这个队列是基于二叉堆来实现的，每次都会把最快要执行的任务放入堆顶(最小堆)。"
tags: ["线程池"]
categories: ["并发编程"]
keywords: ["线程池","基础"]
---
### 问题
* ScheduledThreadPoolExecutor 与 Timer 执行定时任务比较，ScheduledThreadPoolExecutor 有什么优势？
* ScheduledThreadPoolExecutor 与 ThreadPoolExecutor 的区别？
* ScheduledThreadPoolExecutor 的任务调度方法有哪些？ 有什么区别？
* ScheduledThreadPoolExecutor 如何实现异步执行并获取结果的？
* ScheduledThreadPoolExecutor 中 DelayedWorkQueue的数据结构是什么？

****
### 简述
>ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor 重用线程池的功能：
* 相比 ThreadPoolExecutor 支持周期性任务的调度
* ScheduledThreadPoolExecutor 基于相对时间
* ScheduledThreadPoolExecutor 会将任务封装成 ScheduledFutureTask， 放入队列中
* ScheduledThreadPoolExecutor 定义了一个延迟队列 DelayedWorkQueue，这个队列是基于二叉堆来实现的，每次都会把最快要执行的任务放入堆顶(最小堆)
* ScheduledFutureTask 继承自 FutureTask，可以通过返回Future对象来异步的获取执行的结果

>对于周期性的任务调度, JDK还有一个实现 Timer
* Timer 是一个单线程模式的，如果在执行某个任务比较耗时，那么会影响接下来的任务
* Timer 是基于绝对时间的，对系统时间比较敏感
* Timer 的任务队列也是使用的二叉堆，处于栈顶的是最快要执行的任务(最小堆)
* Timer 作为单线程，如果执行任务的过程中发生错误，那么会导致接下来的任务无法执行
****
### 源码分析
#### 类图
![](/img/blog/线程池/ScheduledThreadPool类图.png)
****
#### 重要参数以及相关方法
```c
//当调用 #shutdown方法的时候是否要继续执行存在的周期性任务  默认是false
private volatile boolean continueExistingPeriodicTasksAfterShutdown;

//当调用 #shutdown方法的时候是否要继续执行延迟任务  默认是true。只有在 调用#shutdownNow方法的时候任务才会被终止
private volatile boolean executeExistingDelayedTasksAfterShutdown = true;

//当取消一个task的时候是否要立马的从工作队列中删除(DelayedWorkQueue) 默认是false
private volatile boolean removeOnCancel = false;

//task的序号 根据FIFO来排序
private static final AtomicLong sequencer = new AtomicLong();

// 接下来的方法就是设置以上的值和获取
public void setContinueExistingPeriodicTasksAfterShutdownPolicy(boolean value) {
        continueExistingPeriodicTasksAfterShutdown = value;
        if (!value && isShutdown())
            onShutdown();
    }

public boolean getContinueExistingPeriodicTasksAfterShutdownPolicy() {
        return continueExistingPeriodicTasksAfterShutdown;
    }

public void setExecuteExistingDelayedTasksAfterShutdownPolicy(boolean value) {
        executeExistingDelayedTasksAfterShutdown = value;
        if (!value && isShutdown())
            onShutdown();
    }

public boolean getExecuteExistingDelayedTasksAfterShutdownPolicy() {
        return executeExistingDelayedTasksAfterShutdown;
    }

public void setRemoveOnCancelPolicy(boolean value) {
        removeOnCancel = value;
    }

public boolean getRemoveOnCancelPolicy() {
        return removeOnCancel;
    }
```
****
#### 构造方法
```c
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory);
    }
public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), handler);
    }
public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
    }
```
**因为 ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor，所以这里都是调用 ThreadPoolExecutor 的构造方法。**
****
### 核心方法
#### schedule 方法
```c
public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        //<1> 封装成 RunnableScheduledFuture 对象
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        //<2> 调用 #delayedExecute 方法执行任务
        delayedExecute(t);
        return t;
    }

public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }
```
**这两个 schedule 方法两个重载方法，只有第一个参数不同。Runnable 和 Callable的区别是：Runnable 没有返回值，无法抛出 Exception**
```c
public interface Runnable {
    public abstract void run();
}
public interface Callable<V> {
    V call() throws Exception;
}
```
**在 schedule 方法中<1>处 通过传入一个任务然后调用 #decorateTask 方法封装成 RunnableScheduledFuture**
```c
//修改或替换用于执行 runnable 的任务。此方法可重写用于管理内部任务的具体类。默认实现只返回给定任务。
protected <V> RunnableScheduledFuture<V> decorateTask(
    Runnable runnable, RunnableScheduledFuture<V> task) {
    return task;
}
//修改或替换用于执行 callable 的任务。此方法可重写用于管理内部任务的具体类。默认实现只返回给定任务。
protected <V> RunnableScheduledFuture<V> decorateTask(
    Callable<V> callable, RunnableScheduledFuture<V> task) {
    return task;
}
```
**然后调用 #delayedExecute 方法来执行延迟任务最后返回 #ScheduledFuture 对象**

****
#### scheduleAtFixedRate 方法
**下一次执行时间相当于是上一次的执行时间加上period，它是采用已固定的频率来执行任务**
```c
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        //<1> 校验
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        //<2> 封装成一个 ScheduledFutureTask
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        //<3> 调用 #delayedExecute()方法来执行延迟任务
        delayedExecute(t);
        return t;
    }
```
****
#### scheduleWithFixedDelay 方法
**与scheduleAtFixedRate方法不同的是，下一次执行时间是上一次任务执行完的系统时间加上period，因而具体执行时间不是固定的，但周期是固定的，是采用相对固定的延迟来执行任务**
```c
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        ////<1> 校验
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        //<2> 封装成一个 ScheduledFutureTask
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        //<3> 调用 #delayedExecute()方法来执行延迟任务
        delayedExecute(t);
        return t;
    }
```
****
#### execute 方法
```c
//立即执行 也就是把 delay 时间置为 0
public void execute(Runnable command) {
        schedule(command, 0, NANOSECONDS);
    }
```
****
#### submit 方法
```c
//也是调用了 schedule 方法
public Future<?> submit(Runnable task) {
        return schedule(task, 0, NANOSECONDS);
    }
public <T> Future<T> submit(Runnable task, T result) {
        return schedule(Executors.callable(task, result), 0, NANOSECONDS);
    }
public <T> Future<T> submit(Callable<T> task) {
        return schedule(task, 0, NANOSECONDS);
    }
```
****
#### shutdown、shutdownNow、getQueue 方法
```c
public void shutdown() {
        super.shutdown();
    }
public List<Runnable> shutdownNow() {
        return super.shutdownNow();
    }
public BlockingQueue<Runnable> getQueue() {
        return super.getQueue();
    }
```
**这些方法都是直接调用父类 ThreadPoolExecutor 中的实现。**
****
#### onShutdown 方法
**复写 ThreadPoolExecutor 中的 onShutdown 方法**
```c
@Override void onShutdown() {
        BlockingQueue<Runnable> q = super.getQueue();
        // 获取在线程池已 shutdown 的情况下是否继续执行现有延迟任务
        boolean keepDelayed =
            getExecuteExistingDelayedTasksAfterShutdownPolicy();
        // 获取在线程池已 shutdown 的情况下是否继续执行现有定期任务
        boolean keepPeriodic =
            getContinueExistingPeriodicTasksAfterShutdownPolicy();
        // 如果在线程池已 shutdown 的情况下不继续执行延迟任务和定期任务
        // 则依次取消任务，否则则根据取消状态来判断
        if (!keepDelayed && !keepPeriodic) {
            for (Object e : q.toArray())
                if (e instanceof RunnableScheduledFuture<?>)
                    ((RunnableScheduledFuture<?>) e).cancel(false);
            q.clear();
        }
        else {
            // Traverse snapshot to avoid iterator exceptions
            for (Object e : q.toArray()) {
                if (e instanceof RunnableScheduledFuture) {
                    RunnableScheduledFuture<?> t =
                        (RunnableScheduledFuture<?>)e;
                    // 如果有在 shutdown 后不继续的延迟任务或周期任务，则从队列中删除并取消任务
                    if ((t.isPeriodic() ? !keepPeriodic : !keepDelayed) ||
                        t.isCancelled()) { // also remove if already cancelled
                        if (q.remove(t))
                            t.cancel(false);
                    }
                }
            }
        }
        tryTerminate();
    }
```
****
#### delayedExecute 方法
**上述的调度方法最终都是调用此方法来执行延迟任务**
```c
private void delayedExecute(RunnableScheduledFuture<?> task) {
        //<1> 判断是否已经处于shutdown状态
        if (isShutdown())
            //<1.1> 如果是的话 拒绝此task
            reject(task);
        else {
            //<2> 往队列中添加此task
            super.getQueue().add(task);
            //<3> 如果这时候被关闭了，根据continueExistingPeriodicTasksAfterShutdown或者executeExistingDelayedTasksAfterShutdown进行判断
            //    是否在shutdown状态下继续完成此task，如果不是。那么移出取消此task
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                //<4> 在threadPoolExecutor 中实现，当前线程少于corePoolSize那就添加一个，如果池子中没有线程那添加一个现车过确保线程池中有一个线程
                ensurePrestart();
        }
    }
```
**也就是说 ScheduledThreadPoolExecutor 最终执行任务其实和还是是让 ThreadPoolExecutor 去执行
ScheduledThreadPoolExecutor 和 ThreadPoolExecutor 的区别在于，ThreadPoolExecutor 是通过不断的从阻塞队列中取出 Runnable 类型的任务并执行。
而 ScheduledThreadPoolExecutor 是不断的从延迟队列 DelayedWorkQueue 中取出 ScheduledFutureTask 类型的任务来执行，来实现周期性调度的功能。**
****
#### ScheduledFutureTask
**ScheduledThreadPoolExecutor 的内部类**
```c
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {
        //任务的序号 不断自增 FIFO
        private final long sequenceNumber;
        //以毫微秒为单位的执行任务的时间
        private long time;
        //如果为正数那么表示fixed-rate，负数表示fixed-delay，0表示不是周期任务
        private final long period;
        RunnableScheduledFuture<V> outerTask = this;
        //进入延迟队列的index，在二叉堆中可以快速找到并且取消 时间复杂度为O(logn)
        int heapIndex;

        //构造
        ScheduledFutureTask(Runnable r, V result, long ns) {
            super(r, result);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }
        ScheduledFutureTask(Runnable r, V result, long ns, long period) {
            super(r, result);
            this.time = ns;
            this.period = period;
            this.sequenceNumber = sequencer.getAndIncrement();
        }
        ScheduledFutureTask(Callable<V> callable, long ns) {
            super(callable);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        //获取该任务还剩多少延迟时间
        public long getDelay(TimeUnit unit) {
            return unit.convert(time - now(), NANOSECONDS);
        } 
        //此compareTo 根据 ScheduledFutureTask 中的time 执行时间来选出最快时间的任务， 用于二叉堆(最小堆)
        public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }

        //判断是否是周期性任务
        public boolean isPeriodic() {
            return period != 0;
        }

        //设置下次执行的时间
        private void setNextRunTime() {
            long p = period;
            if (p > 0)
                time += p;
            else
                time = triggerTime(-p);
        }

        //取消任务
        public boolean cancel(boolean mayInterruptIfRunning) {
            boolean cancelled = super.cancel(mayInterruptIfRunning);
            if (cancelled && removeOnCancel && heapIndex >= 0)
                remove(this);
            return cancelled;
        }

        //对标Runnable中的run方法
        public void run() {
            boolean periodic = isPeriodic();
            //<1> 根据状态判断是否满足执行的条件，不满足就取消任务
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            //<2>  如果不是周期性任务，调用FutureTask中的run方法执行
            else if (!periodic)
                ScheduledFutureTask.super.run();
            //<3> 如果是周期性任务，调用FutureTask中的runAndReset方法执
            else if (ScheduledFutureTask.super.runAndReset()) {
                //<3.1> 计算下次执行该任务的时间
                setNextRunTime();
                //<3.2> 重复执行任务
                reExecutePeriodic(outerTask);
            }
        }
    }
```
****
#### DelayedWorkQueue
**ScheduledThreadPoolExecutor之所以要自己实现阻塞的工作队列，在执行定时任务的时候，每个任务的执行时间都不同，
所以DelayedWorkQueue的工作就是按照执行时间的升序来排列，执行时间距离当前时间越近的任务在队列的前面,那这种优先队列的模式。
它可以保证每次出队的任务都是当前队列中执行时间最靠前的，由于它是基于堆结构的队列，堆结构在执行插入和删除操作时的最坏时间复杂度是 O(logN)。**

****
#### 参数
```c
//初始容量
private static final int INITIAL_CAPACITY = 16;
//RunnableScheduledFuture类型的数组
private RunnableScheduledFuture<?>[] queue = new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
//锁
private final ReentrantLock lock = new ReentrantLock();
//数量
private int size = 0;
// leader线程
private Thread leader = null;
// 条件队列：当较新的任务在队列的头部可用时，或者新线程可能需要成为leader，则通过该条件发出信号
private final Condition available = lock.newCondition();
```
**所有线程会有三种身份中的一种：leader和follower，以及一个干活中的状态：proccesser。它的基本原则就是，永远最多只有一个leader。而所有follower都在等待成为leader。线程池启动时会自动产生一个Leader负责等待网络IO事件，当有一个事件产生时，Leader线程首先通知一个Follower线程将其提拔为新的Leader，然后自己就去干活了，去处理这个网络事件，处理完毕后加入Follower线程等待队列，等待下次成为Leader。这种方法可以增强CPU高速缓存相似性，及消除动态内存分配和线程间的数据交换。**

[参考自https://blog.csdn.net/goldlevi/article/details/7705180](https://blog.csdn.net/goldlevi/article/details/7705180)
****
#### offer 方法
```c
public boolean offer(Runnable x) {
            if (x == null)
                throw new NullPointerException();
            RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
            final ReentrantLock lock = this.lock;
            // 加锁
            lock.lock();
            try {
                int i = size;
                // queue是一个RunnableScheduledFuture类型的数组，如果容量不够需要扩容
                if (i >= queue.length)
                    grow();
                size = i + 1;
                // i == 0 说明堆中还没有数据
                if (i == 0) {
                    queue[0] = e;
                    setIndex(e, 0);
                } else {
                    // i != 0 时，自下而上进行堆化
                    siftUp(i, e);
                }
                // 如果传入的任务已经是队列的第一个节点了，这时available需要发出信号
                if (queue[0] == e) {
                    // leader设置为null为了使在take方法中的线程在通过available.signal();后会执行available.awaitNanos(delay);
                    leader = null;
                    available.signal();
                }
            } finally {
                lock.unlock();
            }
            return true;
        }
```
****
#### take 方法
```c
public RunnableScheduledFuture<?> take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    RunnableScheduledFuture<?> first = queue[0];
                    // 当堆中没有数据 等待元素插入后被唤醒
                    if (first == null)
                        available.await();
                    else {
                        // 计算当前时间到执行时间的时间间隔
                        long delay = first.getDelay(NANOSECONDS);
                        if (delay <= 0)
                            return finishPoll(first);
                        first = null; // don't retain ref while waiting
                        // leader不为空，阻塞线程
                        if (leader != null)
                            available.await();
                        else {
                            // leader为空，则把leader设置为当前线程，
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                                // 阻塞到执行时间
                                available.awaitNanos(delay);
                            } finally {
                                // 设置leader = null，让其他线程执行available.awaitNanos(delay);
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                // 如果leader不为空，则说明leader的线程正在执行available.awaitNanos(delay);
                // 如果queue[0] == null，说明队列为空
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }
```
> ThreadPoolExecutor中，getTask方法，工作线程会循环地从workQueue中取任务。但定时任务却不同，因为如果一旦getTask方法取出了任务就开始执行了，而这时可能还没有到执行的时间，所以在take方法中，要保证只有在到指定的执行时间的时候任务才可以被取走。

> 这里的leader是为了减少不必要的定时等待，当一个线程成为leader时，它只等待下一个节点的时间间隔，但其它线程无限期等待。 leader线程必须在从take（）或poll（）返回之前signal其它线程，除非其他线程成为了leader。

> 试想一下，如果没有leader，那么在执行take时，都要执行available.awaitNanos(delay)，假设当前线程执行了该段代码，这时还没有signal，第二个线程也执行了该段代码，则第二个线程也要被阻塞。多个这时执行该段代码是没有作用的，因为只能有一个线程会从take中返回queue[0]（因为有lock），其他线程这时再返回for循环执行时取的queue[0]，已经不是之前的queue[0]了，然后又要继续阻塞。
所以，为了不让多个线程频繁的做无用的定时等待，这里增加了leader，如果leader不为空，则说明队列中第一个节点已经在等待出队，这时其它的线程会一直阻塞，减少了无用的阻塞（注意，在finally中调用了signal()来唤醒一个线程，而不是signalAll()）。

其他方法和PriorityQueue 以及 DelayQueue都类似。

### 总结 
* **ScheduledThreadPoolExecutor 继承自 ThreadPoolExecutor，所以它也是一个线程池，也有 corePoolSize 和 workQueue，ScheduledThreadPoolExecutor 特殊的地方在于，自己实现了优先工作队列 DelayedWorkQueue**
* **ScheduledThreadPoolExecutor 实现了 ScheduledExecutorService，所以就有了任务调度的方法，如schedule，scheduleAtFixedRate 和 scheduleWithFixedDelay**
* **内部类ScheduledFutureTask 继承自 FutureTask，实现了任务的异步执行并且可以获取返回结果。同时也实现了Delayed接口，可以通过getDelay方法获取将要执行的时间间隔**
* **周期任务的执行其实是调用了FutureTask类中的runAndReset方法，每次执行完设置下一次的执行时间周而复始**
* **DelayedWorkQueue的数据结构，它是一个基于最小堆结构的优先队列，并且每次出队时能够保证取出的任务是当前队列中下次执行时间最小的任务。堆结构只保证了子节点的值要比父节点的值要大，但是不保证在整个堆中都是有序的**