---
title: "ThreadPoolExecutor 源码分析"
date: 2020-01-18T19:12:42+08:00
draft: false
banner: "/img/blog/banners/006tNbRwly1fyb51imvdpj31420u0hdt.jpg"
author: "Siran"
summary: "ThreadPoolExecutor中常用参数有哪些，作用是什么？任务提交后，ThreadPoolExecutor会按照什么策略去创建线程用于执行提交任务？"
tags: ["线程池"]
categories: ["线程池"]
keywords: ["线程池","基础"]
---

### 问题
* ThreadPoolExecutor中常用参数有哪些，作用是什么？任务提交后，ThreadPoolExecutor会按照什么策略去创建线程用于执行提交任务？
* ThreadPoolExecutor有哪些状态，状态之间流转是什么样子的？
* ThreadPoolExecutor中的线程哪个时间点被创建？是任务提交后吗？可以在任务提交前创建吗？
* ThreadPoolExecutor中创建的线程哪个时间被启动？ThreadPoolExecutor竟然是线程池那么他是如何做到重复利用线程的？
* ThreadPoolExecutor中创建的同一个线程同一时刻能执行多个任务吗？
* 如果不能是通过什么机制保证ThreadPoolExecutor中的同一个线程只能执行完一个任务，才会机会去执行另一个任务？
* ThreadPoolExecutor中关闲线程池的方法shutdown与shutdownNow的区别是什么？
* 通过submit方法向ThreadPoolExecutor提交任务后，当所有的任务都执行完后不调用shutdown或shutdownNow方法会有问题吗？
* ThreadPoolExecutor有没有提供扩展点，方便在任务执行前或执行后做一些事情？
* 在runWorker方法中，执行任务时对Worker对象w进行了lock操作，为什么要在执行任务的时候对每个工作线程都加锁呢？

****
### 简述
>正常情况下，web开发中，每次接受一个请求，那么就会创建一个线程进行处理。
但是如果并发过多，执行速度很快，频繁的进行线程的创建和销毁，这是非常耗资源的，也会导致整个系统效率降低，服务器的cpu飙升。因为根据linux目前的线程实现Pthread，是与Java中的线程一一对应的，那么通过cpu频繁的创建线程，非常耗资源。
那么如果我请求结束后我不销毁线程进行复用那么就可以避免频繁的创建线程，这时候就出现了线程池。

**线程池的好处:**
>降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
提高响应速度。当任务达到时，任务可以不需要等待线程创建就立马执行
提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

**线程池的类图：**
![](/img/blog/线程池/线程池类图.png)
*****
### 源码分析
#### 顶级接口 Executor 提供一个执行的方法 execute：
*****
```c
public interface Executor {
    void execute(Runnable command);
}
//可以实现Executor接口调用execute来执行一个线程
class DirectExecutor implements Executor {
   public void execute(Runnable r) {
   r.run();
}
```
*****

#### ExecutorService 接口：
**继承了Executor，提供了管理终止的方法，以及可为跟踪一个或多个异步执行状况而生成Future的方法。增加了shutdown()，shutDownNow()，invokeAll()，invokeAny() 和submit()等方法。如果需要支持即时关闭，也就是shutDownNow()，则任务需要正确处理中断。**
```c
public interface ExecutorService extends Executor {
    void shutdown();
​
    List<Runnable> shutdownNow();
​
    boolean isShutdown();
​
    boolean isTerminated();
​
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
​
    <T> Future<T> submit(Callable<T> task);
​
    <T> Future<T> submit(Runnable task, T result);
​
    Future<?> submit(Runnable task);
​
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
​
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
​
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
​
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
*****

#### ScheduledExecutorService 接口: 
**继承 ExecutorService，用于扩展ExecutorService接口并增加schedule方法。调用schedule方法可以在指定的延时后执行一个Runnable或者Callable任务。还定义了按照时间间隔执行任务的scheduleAtFixedRate()方法和scheduleWithFixedDelay()方法。**
```c
public interface ScheduledExecutorService extends ExecutorService {
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);
​
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);
​
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);
​
  
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
​
}
```
*****

#### AbstractExecutorService 
**它是一个抽象类，实现了 ExecutorService 接口。并定义好了submit、shutdown、invoke等方法。
ThreadPoolExecutor 继承自AbstractExecutorService。**
*****

#### 重要字段：
**ctl 是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段，它包含两部分信息：线程池的运行状态(runState) 和线程池内有效线程的数量(workerCount),使用Integer类型来保存，高3位保存runState，低29位来保存workerCount。COUNT_BITS 就是29 ，CAPACITY 就是1左移29位减一这个常量表示workerCount的上限值，大约是5亿**
```c
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
​
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```
*****
#### ctl的相关方法：
```c
//获取运行状态    
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//获取有效线程数量
private static int workerCountOf(int c)  { return c & CAPACITY; }
//获取运行状态和有效线程的数量
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
*****
#### 五种状态：
>`RUNNING`：接受新任务并且处理队列里的任务
>
>`SHUTDOWN`：不接受新任务，处理队列里的任务；Running状态时调用shutdown()方法进入该状态（finalize()方法中也会隐式的调用shutdown方法）
>
>`STOP`：不接受新任务，不处理队列里的任务 中断正在运行的任务；Running状态 或者 Shutdown状态 调用shutdownNow()会进入此状态
>
>`TIDYING`：所有任务终止，workCount(有效线程数)为0；进入此状态会调用terminated()方法进入terminated状态
>
>`TERMINATED`：在terminated()方法执行完后进入该状态，默认什么都不做。

![](/img/blog/线程池/线程池状态.png)

*****
#### 构造方法：
```c
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
****
#### 构造器中的字段含义：
* **`corePoolSize`：核心线程数量**
* **`maximumPoolSize`：最大线程数量   当有新任务在execute()方法提交时，会执行以下判断：**
    1. 如果当前线程池中的线程数少于corePoolSize的数量，不管其他的线程是否空闲都会创建新线程来执行新任务
    2. 如果超过corePoolSize的数量但是小于maximumPoolSize的数量，只有等待队列满了才会创建新的线程
    3. 如果设置的corePoolSize和maximumPoolSize数量相同，则创建的线程池是固定的，这时如果有新任务提交，若workQueue未满，则将请求放入workQueue中，等待有空闲的线程从workQueue中取出任务并处理
    4. 如果运行的线程数量大于等于maximumPoolSize，这时如果workQueue满了，则通过handler所指定的拒绝策略来处理任务默认时
* **`keepAliveTime`：线程池中允许线程的空闲时间，如果当前线程池中的线程超过了corePoolSize，那么空闲时间超过keepAliveTime就会被terminated**
* **`workQueue`：等待队列，当任务提交时，如果线程池中的线程数量大于等于corePoolSize的时候，把该任务封装成一个Worker对象放入等待队列**
    1. 直接切换：常用队列SynchronousQueue
    2. 无界队列：常用LinkedBlockingQueue，如果使用这种方式，那么线程池中能够创建的最大线程数就是corePoolSize，而maximumPoolSize就不会起作用了。当线程池中所有的核心线程都是RUNNING状态时，这时一个新的任务提交就会放入等待队列中。 
    3. 有界队列：常用ArrayBlockingQueue
* **`threadFactory`：创建线程的工厂，默认使用Executors.defaultThreadFactory() 来创建线程。使用默认的ThreadFactory来创建线程时，会使新创建的线程具有相同的NORM_PRIORITY优先级并且是非守护线程，同时也设置了线程的名称。**
* **`handler`：拒绝策略，如果阻塞队列满了并且没有空闲的线程，这时如果继续提交任务，就需要采取一种策略处理该任务。线程池提供了4种策略：**
    1. abortPolicy：直接抛出异常，默认策略
    2. CallerRunsPolicy：用掉用者所在的线程来执行任务
    3. DiscardOldestPolicy：丢弃堵塞队列中最靠前的任务，并执行当前任务
    4. DiscardPolicy：直接丢弃

****
#### 主要方法：
```c
//execute方法用来提交任务：
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        // ctl 记录了线程池的runState和workCount
        int c = ctl.get();
        // 1. 如果当前线程小于 corePoolSize 通过调用addWorkd()方法尝试去创建一个新的线程，在addWork()方法中会检查runState 和workCount，成功则直接return失败则返回false
        // 2. 如果失败在获取ctl 然后通过isRunning()判断是否是running状态 并且 可以成功入队，则继续检查是否是running状态如果不是running状态了则调用remove方法从队列中移出并且调用reject拒绝任务
        //    反之判断workCount的数量是否==0 如果是的话调用addWorker() 创建线程   
        // 3. 如果入队列失败 则尝试调用addWorkd()方法创建线程执行任务，如果失败则调用reject()拒绝任务。
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

>简单来说，在执行execute()方法时如果状态一直是RUNNING时，的执行过程如下：
>
>如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务；
>
>如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中；
>
>如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务；
>
>如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。
>
>这里要注意一下addWorker(null, false);，也就是创建一个线程，但并没有传入任务，因为任务已经被添加到workQueue中了，所以worker在执行的时候，会直接从workQueue中获取任务。所以，在workerCountOf(recheck) == 0时执行addWorker(null, false);也是为了保证线程池在RUNNING状态下必须要有一个线程来执行任务。
>
****
#### 执行流程：
![](/img/blog/线程池/线程池执行流程.png)

****
#### addWorkder方法
**主要工作是在线程池中创建一个新的线程并执行，firstTask参数 用于指定新增的线程执行的第一个任务，core参数为true表示在新增线程时会判断当前活动线程数是否少于corePoolSize，false表示新增线程前需要判断当前活动线程数是否少于maximumPoolSize，代码如下：**
```c
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            //获取运行状态
            int c = ctl.get();
            int rs = runStateOf(c);
​
            /*
              如果rs >= SHUTDOWN，则表示不接受新任务
              接着判断()中的条件，只要满足其一，则直接返回fasle
              1.rs ==SHUTDOWN 此状态在上面讲过表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
              2.firstTask 是否为空
              3.等待队列不为空
              因为当你runState==Shutdown的情况，不接受新的任务，所以在firstTask不为空的时候会返回fasle
              如果firstTask 为空，并且队列也是空 则返回fasle 因为队列中已经没有任务了，不需要在添加线程了
             **/
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
​
            for (;;) {
                //获取线程数
                int wc = workerCountOf(c);
                //大于最大容量5亿 返回false
                //这里的core 是addWorker方法的第二个参数 ，true为和corePoolSize比较， 反之和max比较
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //增加workerCount，如果成功 跳出
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //增加失败，重新获取ctl的值
                c = ctl.get();  // Re-read ctl
                //比较状态，如果不相等说明线程池状态发生了改变，返回第一个for循环继续执行
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
​
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 根据firstTask来创建Worker对象
            w = new Worker(firstTask);
            // 每一个Worker对象都会创建一个线程
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    // rs < SHUTDOWN表示是RUNNING状态；
                    // 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程。
                    // 因为在SHUTDOWN时不会在添加新的任务，但还是会执行workQueue中的任务
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        // largestPoolSize记录着线程池中出现过的最大线程数量
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
**这里的t.start()这个语句，启动时会调用Worker类中的run方法，Worker本身实现了Runnable接口，所以一个Worker类型的对象也是一个线程。**
****
#### Workder类
```c
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;
​
        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;
​
        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
​
        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }
​
        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.
​
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
​
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
​
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
​
        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }
​
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
>Worker类继承AQS，实现runnable，在调用Worker类的构造方法的时候通过getThreadFactory().newThread(this) 来创建线程，因为Worker本身继承了Runnable接口，也就是一个线程，所以一个Worker对象在启动的时候会调用Worker类中的run方法。
>
>Worker 继承AQS,实现tryAcquire方法和tryRelease方法的主要目的是要判断线程是否可以被中断以及是否空闲：
>
>lock方法一旦获取了独占锁，那么就表示线程正在执行中
>
>如果正在执行任务，则不应该中断线程
>
>如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断；
>
>线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；
>
>之所以设置为不可重入，是因为我们不希望任务在调用像setCorePoolSize这样的线程池控制方法时重新获取锁。如果使用ReentrantLock，它是可重入的，这样如果在任务中调用了如setCorePoolSize这类线程池控制的方法，会中断正在运行的线程。
>
>此外，在构造方法中执行了setState(-1);，把state变量设置为-1，为什么这么做呢？是因为AQS中默认的state是0，如果刚创建了一个Worker对象，还没有执行任务时，这时就不应该被中断，看一下tryAquire方法：
```c
protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```
**tryAcquire方法是根据state是否是0来判断的，所以，setState(-1);将state设置为-1是为了禁止在执行任务前对线程进行中断。**
****
#### runWorker方法
**在Worker类中的run()调用了runWorker()方法来来执行任务。**
```c
public void run() {
            runWorker(this);
}
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```
**上述方法中这个判断** 
```c
if ((runStateAtLeast(ctl.get(), STOP) ||(Thread.interrupted() &&runStateAtLeast(ctl.get(), STOP))) &&!wt.isInterrupted())
private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
}
```
>如果线程正在停止，那么要保证当前线程是中断状态；如果不是的话则要保证当前线程不是中断状态
>
>这里要考虑在执行该if语句期间可能也执行了shutdownNow方法，shutdownNow方法会把状态设置为STOP
>
>STOP状态要中断线程池中的所有线程，而这里使用Thread.interrupted()来判断是否中断是为了确保在RUNNING或者SHUTDOWN状态时线程是非中断状态的，因为Thread.interrupted()方法会复位中断的状态。
>
>上述方法中的beforeExecute 和 afterExecute方法可需要子类来实现分别是在任务前后执行
****
#### getTask方法
```c
private Runnable getTask() {
        // timeOut变量的值表示上次从阻塞队列中取任务时是否超时
        boolean timedOut = false; // Did the last poll() time out?
​
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
​
            // Check if queue empty only if necessary.
            //如果线程池状态非running 并且 状态是否正在stop 或者 等待队列为空
            //则将workerCount 减1 并返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
​
            int wc = workerCountOf(c);
​
            // Are workers subject to culling?
            //timed变量 用于判断是否要进行超时判断  allowCoreThreadTimeOut默认是false，也就是核心线程不允许进行超时；
            //wc > corePoolSize;  超过了corePoolSize数量就要进行超时判断
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
​
            /*
             * wc > maximumPoolSize 的情况是因为可能在此方法执行阶段同时执行了setMaximumPoolSize方法；
             * timed && timedOut 如果为true，表示当前操作需要进行超时控制，并且上次从阻塞队列中获取任务发生了超时
             * 接下来判断，如果有效线程数量大于1，或者阻塞队列是空的，那么尝试将workerCount减1；
             * 如果减1失败，则返回重试。
             * 如果wc == 1时，也就说明当前线程是线程池中唯一的一个线程了。
             */
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
​
            try {
                 /*
                  * 根据timed来判断，如果为true，则通过阻塞队列的poll方法进行超时控制，如果在keepAliveTime时间内没有获取到任务，则返回null；
                  * 否则通过take方法，如果这时队列为空，则take方法会阻塞直到队列不为空。
                  */
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                // 如果 r == null，说明已经超时，timedOut设置为true
                timedOut = true;
            } catch (InterruptedException retry) {
                // 如果获取任务时当前线程发生了中断，则设置timedOut为false并返回循环重试
                timedOut = false;
            }
        }
    }
```
>这里重要的地方是第二个if判断，目的是控制线程池的有效线程数量。由上文中的分析可以知道，在执行execute方法时，如果当前线程池的线程数量超过了corePoolSize且小于maximumPoolSize，并且workQueue已满时，则可以增加工作线程，但这时如果超时没有获取到任务，也就是timedOut为true的情况，说明workQueue已经为空了，也就说明了当前线程池中不需要那么多线程来执行任务了，可以把多于corePoolSize数量的线程销毁掉，保持线程数量在corePoolSize即可。
>
>什么时候会销毁？当然是runWorker方法执行完之后，也就是Worker中的run方法执行完，由JVM自动回收。
>
>getTask方法返回null时，在runWorker方法中会跳出while循环，然后会执行processWorkerExit方法。

****
#### processWorkerExit方法
```c
private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 如果completedAbruptly值为true，则说明线程执行时出现了异常，需要将workerCount减1；
        // 如果线程执行时没有出现异常，说明在getTask()方法中已经已经对workerCount进行了减1操作，这里就不必再减了。
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();
​
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //统计完成的任务数
            completedTaskCount += w.completedTasks;
            // 从workers中移除，也就表示着从线程池中移除了一个工作线程
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
​
        // 根据线程池状态进行判断是否结束线程池
        tryTerminate();
​
        int c = ctl.get();
        //当线程池是RUNNING或SHUTDOWN状态时
        if (runStateLessThan(c, STOP)) { 
            if (!completedAbruptly) {
                //如果allowCoreThreadTimeOut=true，并且等待队列有任务，至少保留一个worker；
                //如果allowCoreThreadTimeOut=false，workerCount不少于corePoolSize。
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            //异常退出 直接addWorker
            addWorker(null, false);
        }
    }
private static boolean runStateLessThan(int c, int s) {
        return c < s;
}
```
**至此，processWorkerExit执行完之后，工作线程被销毁，以上就是整个工作线程的生命周期，从execute方法开始，Worker使用ThreadFactory创建新的工作线程，runWorker通过getTask获取任务，然后执行任务，如果getTask返回null，进入processWorkerExit方法，整个线程结束，如图所示：**
![](/img/blog/线程池/线程池执行流程2.png)

****
#### tryTerminate方法  根据线程池状态进行判断是否结束线程池
```c
final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            /*
             * 当前线程池的状态为以下几种情况时，直接返回：
             * 1. RUNNING，因为还在运行中，不能停止；
             * 2. TIDYING或TERMINATED，因为线程池中已经没有正在运行的线程了；
             * 3. SHUTDOWN并且等待队列非空，这时要执行完workQueue中的task；
             */
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 如果线程数量不为0，则中断一个空闲的工作线程，并返回
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
​
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 这里尝试设置状态为TIDYING，如果设置成功，则调用terminated方法
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        // terminated方法默认什么都不做，留给子类实现
                        terminated();
                    } finally {
                        // 设置状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
}
```
****
#### interruptIdleWorkers方法
```c
private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }
private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //这个workers 是一个HashSet 
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
**上述方法持锁的原因是因为HashSet是非安全的**
****
#### shutdown方法
```c
//将线程池切换到SHUTDOWN状态，并调用interruptIdleWorkers方法请求中断所有空闲的worker，最后调用tryTerminate尝试结束线程池。
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 安全策略判断
            checkShutdownAccess();
            // 切换状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
            // 中断空闲线程
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        // 尝试结束线程池
        tryTerminate();
    }
```
****
#### shutdownNow方法
```c
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            // 中断所有工作线程，无论是否空闲
            interruptWorkers();
            // 取出队列中没有被执行的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
    
shutdownNow方法与shutdown方法类似，不同的地方在于：
设置状态为STOP；
中断所有工作线程，无论是否是空闲的；
取出阻塞队列中没有被执行的任务并返回。

### 线程池的监控：
**通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用**

* `getTaskCount`：线程池已经执行的和未执行的任务总数
* `getCompletedTaskCount`：线程池已完成的任务数量，该值小于等于taskCount
* `getLargestPoolSize`：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了maximumPoolSize 
* `getPoolSize`：线程池当前的线程数量
* `getActiveCount`：当前线程池中正在执行任务的线程数量。 

**通过这些方法，可以对线程池进行监控，在ThreadPoolExecutor类中提供了几个空方法，如beforeExecute方法，afterExecute方法和terminated方法，可以扩展这些方法在执行前或执行后增加一些新的操作，例如统计线程池的执行任务的时间等，可以继承自ThreadPoolExecutor来进行扩展。**

****
### 总结
1. **一般在提交任务后，会根据传ThreadFactory去创建线程，但当线程池中的线程数已为corePoolSize，或为maximumPoolSize时会不创建线程。可以通过prestartCoreThread或者prestartAllCoreThreads方法预先创建核心线程**
2. **在addWorker方法中创建线程，创建完成后就会被启动。线程池中创建的线程封装到worker对象，而work类又实现了Runnable接口，线程池中的线程又引用了worker。当线程被start后实际就有机会等待操作系统调度执行worker类的run方法**
3. **一旦通过ThreadFactory创建好线程后，就会将创建的线程封装到worker对象中，同时启动该线程。启动的线程会不断的从workerQueue中取出任务来执行(getTask方法)**
4. **一个线程不可以同时执行多个任务，只有一个任务执行完时才能去执行另一个任务。因为ThreadFactory创建线程后封装到worker对象中，worker对象继承了AQS，具有了独占锁的语义，每次执行任务都会先lock操作，执行完后unlock。所以同一个线程执行完后再去执行另一个任务**
5. **如果没设置核心线程允许超时将会有问题。在从workerQUeue中获取任务时，采用的堵塞获取方式等待任务到来但是如果不调用shutdown或shutdownNow方法，核心线程由于在getTask方法调用BlockingQueue.take方法获取任务而处于一直被阻塞挂起状态。核心线程将永远处于Blocking的状态，导致内存泄漏，主线程也无法退出，除非强制kill。**