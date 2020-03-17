---
title: "FutureTask 源码分析"
date: 2020-02-15T19:12:42+08:00
draft: false
banner: "/img/blog/banners/006tNbRwly1fug0hms6vej31jk15ou0x.jpg"
author: "Siran"
summary: "FutureTask 是一个可以取消的异步计算任务，实现Future，Runnable。提供超时控制、可以获取线程执行后的返回结果、可以取消。"
tags: ["线程池"]
categories: ["并发编程"]
keywords: ["线程池","基础"]
---

### 问题
1. FutureTask 是线程安全的吗？
2. FutureTask 状态是如何转变的？
3. 获取FutureTask的result是立即返回的吗？如果不是，那么内部的流程是什么？
****
### 简述
>FutureTask 是一个可以取消的异步计算任务，实现Future，Runnable。提供超时控制、可以获取线程执行后的返回结果、可以取消。

![](/img/blog/线程池/Future类图.png)
****

### 源码分析
#### FutureTask的状态
当刚创建FutureTask的时候它的状态是NEW，在运行时状态会发生转换，有以下四中状态：
* NEW -> COMPLETING -> NORMAL ： 正常执行完
* NEW -> COMPLETING -> EXCEPTIONAL ： 执行过程中出现异常
* NEW -> CANCELLED ： 执行前被取消了
* NEW -> INTERRUPTING -> INTERRUPTED ：执行中被中断了
****
#### 主要参数
```c
    //需要执行的任务，因为futureTask是基于Callable实现的，而Callable可以理解为有返回值的Runnable。当执行后会置为null
    private Callable<V> callable;
    //执行任务要返回的结果或要从get()抛出的异常
    private Object outcome; // non-volatile, protected by state reads/writes
    //运行这个任务的线程
    private volatile Thread runner;
    //等待的线程
    private volatile WaitNode waiters;
```
****
#### 构造函数
```c
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```
**在这两个构造函数中 当传入的是Runnable，需要多传一个result。因为Runnable 是不带返回值的，需要调用#Executors.callable()方法封装成callable
这时候的FutureTask的状态是NEW**
****
#### run 方法
**因为FutureTask实现了Runnable，所以要实现#run()方法**
```c
public void run() {
        //<1> 判断状态，因为一个新的任务的初始状态是NEW，如果不是NEW说明已经被执行了。那么直接返回
        //    通过cas来设置当前执行该任务的线程，这个runner 是运行这个任务所使用的线程，如果无法设置成功那么说明也已经被执行了，直接返回
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            //执行的任务
            Callable<V> c = callable;
            //<2> 再次校验
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    //<3> 调用callable的#call方法执行任务，可以类比为Runnable的#run方法
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //如果发生错误则设置异常
                    setException(ex);
                }
                //<4> 设置返回值
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            // 无论是否执行成功，把runner设置为null
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            // 如果被中断，则说明调用的cancel(true)，
            // 这里要保证在cancel方法中把state设置为INTERRUPTED
            // 否则可能在cancel方法中还没执行中断，造成中断的泄露
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
****
#### setException 方法
```c
protected void setException(Throwable t) {
        //<1> cas 设置该任务的状态为 COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //<2> 在上面重要参数的时候讲过，outcome是用于存放正常的返回值和异常的
            outcome = t;
            //<3> 在把这个状态设置为 EXCEPTIONAL
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            //<4> 最后调用此方法，此方法正如字面意思 完成这次任务：删除并通知所有等待的线程，调用done()，并将可调用的部分空出。
            finishCompletion();
        }
    }
```
****
#### set方法
```c
protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```
**这个方法与setException类似，只不过这里的outcome是返回结果对象，状态先设置为COMPLETING，然后再设置为NORMAL。正常情况**
****
#### handlePossibleCancellationInterrupt 方法
```c
private void handlePossibleCancellationInterrupt(int s) {
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt
    }
```
**handlePossibleCancellationInterrupt方法要确保cancel(true)产生的中断发生在run或runAndReset方法执行的过程中。这里会循环的调用Thread.yield()来确保状态在cancel方法中被设置为INTERRUPTED。**
****
#### finishCompletion 方法
```c
private void finishCompletion() {
        // assert state > COMPLETING;
        // 执行该方法时state必须大于COMPLETING
        // 逐个唤醒waiters中的线程
        for (WaitNode q; (q = waiters) != null;) {
            //<1> 设置栈顶节点为null
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    //<2> 唤醒线程
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    //便于gc
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }
        //勾子方法
        done();

        callable = null;        // to reduce footprint
    }
```
**在调用#get()方法的时候，如果任务还没有执行完成(不管这个任务是正常还是异常)。那么#get()方法会调用#awaitDown()方法把调用线程放入waiter中进行等待，
那么假如任务执行完成了那么会调用#finishCompletion()方法来唤醒在waiter中等待的线程。**
****
#### get 方法
```c
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //<1> 如果当前任务正在运行，那么会调用#awaitDone() 方法把调用线程放入waiter里进行等待
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
//重载方法，此方法具有超时功能，超时还没任务还没完成的话会抛出TimeoutException
public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }
```
****
#### awaitDown 方法
**awaitDone方法是根据状态来判断是否能够返回结果，如果任务还未执行完毕，要添加到waiters中并阻塞，否则返回状态。**
```c
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        //<1> 如果设置了超时时间，那么这里会进行计算超时时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false; 
        //死循环
        for (;;) {
            //<2> 如果线程被中断了，在waiter中移出节点，并抛出中断异常。
            //    这里的#interrupted()方法最终是调用的isInterrupted(true) 判断是否线程中断如果中断则清除中断标记
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            //<3> 如果任务执行完毕并且设置了最终状态或者被取消，则返回
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            //<4> 此时任务在执行中，则让其他线程执行。为了改变这个任务的状态
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            //<5> 创建一个WaitNode
            else if (q == null)
                q = new WaitNode();
            //<6> 放入waiter中，这里只会放入一次，通过queued来控制。
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            //<7> 是否设置超时，是的话进行超时的计算。如果超时了从waiter中移除节点，并反正状态(TimeOutException)
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                //等待 可以类比为wait
                LockSupport.parkNanos(this, nanos);
            }
            else
                //等待 可以类比为wait
                LockSupport.park(this);
        }
    }
```
****
#### removeWaiter 方法
```c
private void removeWaiter(WaitNode node) {
    if (node != null) {
        //<1> 将thread设置为null是因为下面要根据thread是否为null判断是否要把node移出
        node.thread = null;
        //<2> 这里自旋保证删除成功
        retry:
        for (;;) {          // restart on removeWaiter race
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                s = q.next;
                // q.thread != null说明该q节点不需要移除
                if (q.thread != null)
                    pred = q;
                //<3> 如果q.thread == null，且pred != null，需要删除q节点
                else if (pred != null) {
                    // 删除q节点
                    pred.next = s;
                    // pred.thread == null时说明在并发情况下被其他线程修改了；
                    // 返回第一个for循环重试
                    if (pred.thread == null) // check for race
                        continue retry;
                }
                //<4> 如果q.thread != null且pred == null，说明q是栈顶节点
                //    设置栈顶元素为s节点，如果失败则返回重试
                else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                      q, s))
                    continue retry;
            }
            break;
        }
    }
}
```
****
#### cancel 方法
**cancel方法用于取消任务，这里可能有两种情况，一种是任务已经执行了，另一种是还未执行**
```c
public boolean cancel(boolean mayInterruptIfRunning) {
        //<1> 如果状态不是NEW，或者设置状态为INTERRUPTING或CANCELLED失败，则取消失败，返回false。
              如果当前任务还没有执行，那么state == NEW，那么会尝试设置状态，如果设置状态失败会返回false，表示取消失败；
              如果当前任务已经被执行了，那么state > NEW，也就是!state == NEW为true，直接返回false。
              也就是说，如果任务一旦开始执行了（state != NEW），那么就不能被取消。
              如果mayInterruptIfRunning为true，要中断当前执行任务的线程。
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            //<2> mayInterruptIfRunning参数表示是否要进行中断
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        //<3> 中断
                        t.interrupt();
                } finally { // final state
                    //<4> 设置中断状态
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```
****
#### report 方法
```c
private V report(int s) throws ExecutionException {
        Object x = outcome;
        //<1> 任务状态如果是正常的直接返回
        if (s == NORMAL)
            return (V)x;
        //<2> 如果>CANCELLED ，要么是取消要么是中断，则抛出取消异常
        if (s >= CANCELLED)
            throw new CancellationException();
        //<3> 这种情况就是在执行过程中发生了错误
        throw new ExecutionException((Throwable)x);
    }
```
****
#### runAndReset 方法
**该方法和run方法类似，区别在于这个方法不会设置任务的执行结果值，所以在正常执行时，不会修改state，除非发生了异常或者中断，最后返回是否正确的执行并复位**
```c
protected boolean runAndReset() {
        //<1> 这里和#run()方法一样
        //    判断状态，因为一个新的任务的初始状态是NEW，如果不是NEW说明已经被执行了。那么直接返回
        //    通过cas来设置当前执行该任务的线程，这个runner 是运行这个任务所使用的线程，如果无法设置成功那么说明也已经被执行了，直接返回
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            //<2> 调用执行
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            runner = null;
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }
```
****
#### WaitNode
**是一个单向链表**
```c
  static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

## 总结 
* **FutureTask 是线程安全的**
* **正常执行没有取消或者发生错误就是 NEW -> COMPLETING -> NORMAL。具体可以看 futureTask的状态部分**
* **在调用#get()方法来用去任务的执行结果，如果该任务没有执行完，则会堵塞当前调用线程直到任务完成被唤醒**
* **如果使用的是#get(long timeout, TimeUnit unit)方法，那么如果在超时时间内没有被唤醒则会抛出TimeOutException**