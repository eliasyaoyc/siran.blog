---
title: "LongAdder 详解"
date: 2020-03-05T11:37:42+08:00
draft: false
banner: "/img/blog/banners/006tKfTcly1g0d0qfcwpfj31400u0u11.jpg"
author: "Siran"
summary: "LongAdder 类是jdk1.8新增的原子类，在多线程环境下，它的性能比普通的Atomic类性能高很多，继承 Striped64，通过Striped64的Cell来实现功能，并且在ConcurrentHashMap中也用了Striped64的Cell。"
tags: ["原子类"]
categories: ["并发编程"]
keywords: ["原子类","Jdk源码","基础"]
---
### 问题
1. LongAdder 的实现原理？
2. LongAdder 与 AtomicLong 差别？
3. LongAdder 是强一致性还是最终一致性的？
4. LongAdder 中的Cell 数组是无限制扩容的吗？
5. LongAdder 是如何消除伪共享的？
****
### 简介
>LongAdder 类是jdk1.8新增的原子类，在多线程环境下，它的性能比普通的Atomic类性能高很多，
继承 Striped64，通过Striped64的Cell来实现功能，并且在ConcurrentHashMap中也用了Striped64的Cell。
****
### 源码分析
#### 实现原理
> 在初始无竞争时，只更新base的值，当有多线程竞争时通过`分段的思想`，让不同的线程更新不同的段，最后把这些段相加(sum)就得到了完整的LongAdder存储的值，最终一致性。

![](/img/blog/原子类/LongAdder.png)
****
#### 内部类    
```c
// Striped64中的内部类，使用@sun.misc.Contended注解，说明里面的值消除伪共享。 
@sun.misc.Contended static final class Cell {
        //每个cell中的值，在LongAdder中就是每次添加的值。使用volatile 来确保内存可见
        volatile long value;
        //构造器
        Cell(long x) { value = x; }
        //cas修改value的值
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe 实力 在静态代码块中实例化
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```
****
#### 主要属性
```c
    //这些属性都是在Striped64中的
    //CPU数
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    //cells数组，存储各个段的值，Cell的数组长度最大是CPU * 2 在下面的#longAccumulate方法中会有体现
    transient volatile Cell[] cells;
    //初始值，如果没有线程修改的话，就会直接通过CAS来修改这个值，不会创建cell
    transient volatile long base;
    //标记是否有线程在扩容或者创建Cell，会通过cas来更新该值
    transient volatile int cellsBusy;
```
****
#### add() 方法
```c
public void add(long x) {
        Cell[] as; //as 是 Striped64中的cells属性 存储各个段的值
        long b, v; //b是Striped64中的base属性 最初的base属性，还没发生竞争; v是当前线程hash到的Cell中存储的值
        int m; //m是cells的长度减1，hash时作为掩码使用
        Cell a; //a是当前线程hash到的Cell

        //条件1： cells不为空说明已经出现过竞争了，cells已经创建了
        //条件2： cas修改base值 失败，说明已经被其他线程改过了，出现竞争
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            //true表示当前竞争还不激烈; false表示竞争激烈，多个线程hash到同一个Cell，可能要扩容
            boolean uncontended = true;
            //条件1：cells为空，说明正在出现竞争，上面是从条件2过来的
            //条件2：从上面的条件2过来，正在出现竞争，还没有创建cell放入数组，所以这个时候会满足此条件
            //条件3：当前线程所在的Cell为空，说明当前线程还没有更新过Cell，应初始化一个Cell
            // 条件4：更新当前线程所在的Cell失败，说明现在竞争很激烈，多个线程hash到了同一个Cell，应扩容
            if (as == null || (m = as.length - 1) < 0 ||
                    // getProbe()方法返回的是线程中的threadLocalRandomProbe字段
                    // 它是通过随机数生成的一个值，对于一个确定的线程这个值是固定的
                    // 除非刻意修改它
                    (a = as[getProbe() & m]) == null ||
                    !(uncontended = a.cas(v = a.value, v + x)))
                // 调用Striped64中的方法处理
                longAccumulate(x, null, uncontended);
        }
    }
```
1. 最初无竞争时只更新base
2. 直到更新base失败时就是出现了竞争，创建cells数组
3. 当多个线程竞争同一个Cell比较激烈时，可能要扩容
****
#### longAccumulate 方法
只有当在上面的#add方法出现了竞争，需要创建或者扩容cell的时候才会调用此方法
```c
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        //<1> 获取存储线程的probe值，会通过#ThreadLocalRandom.current()方法随机生成一个，之后就不会改变除非自己手动去修改
        if ((h = getProbe()) == 0) {
            //如果没有初始化，那么进行初始化
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            // 还没初始化，说明竞争的不激烈
            wasUncontended = true;
        }
        //是否出现碰撞
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            //<2> 如果Cells数组已经初始化
            if ((as = cells) != null && (n = as.length) > 0) {
                //<2.1> 当前线程进行hash获取在Cells数组中的值，判断是否为空，如果为空的话，进行初始化当前线程的cell然后塞入Cells数组中
                if ((a = as[(n - 1) & h]) == null) {
                    //判断是否有竞争，其他线程在创建或者扩容Cells
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        //创建cell
                        Cell r = new Cell(x);   // Optimistically create
                        //再次判断cellsBusy的值，并且cas的修改 意味着获取锁
                        if (cellsBusy == 0 && casCellsBusy()) {
                            //标记是否创建成功
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                //重新获取Cells数组，找到当前线程hash到Cells数组中的位置。如果为null的话，则加入Cells数组中，create标记为true
                                //note : 这里重新获取Cells数组的原因是因为可能有其他线程已经扩容了或者修改了里面的值
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                //释放锁
                                cellsBusy = 0;
                            }
                            //创建成功直接返回
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    //标记没有发生冲突
                    collide = false;
                }
                // 这里对应了<2.1> 如果当前线程hash到Cells数组的位置不为null，并且wasUncontended的值为false 出现了竞争。则更新失败，继续自旋
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                // 上面的else if 中如果wasUncontended为true 则没有出现竞争，尝试修改当前线程所在的cell的值，成功直接返回
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                // 查看是否要进行扩容，因为每次扩容是 * 2 所以当到了cpu的核数就不会在扩容了
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                //如果上上个elseif 修改cell的值失败了 且上个条件不成立  出现了冲突，collide值改为true
                else if (!collide)
                    collide = true;
                // 到这里就说明已经出现了冲突了，尝试获取占有锁，并且扩容
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        //检查是否有其它线程已经扩容过了
                        if (cells == as) {      // Expand table unless stale
                            //扩大2倍 旧数组拷贝到新数组中，重新赋值
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        //释放锁
                        cellsBusy = 0;
                    }
                    //解决了冲突，继续尝试
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                //更新失败或者达到了CPU核心数，重新生成probe，并重试
                h = advanceProbe(h);
            }

            //<3> 对应<2> Cell数组还没初始化，cas修改cellsBusy的值，相当于是一个独占锁
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                //初始化是否成功
                boolean init = false;
                try {                           // Initialize table
                    //检测是否有其它线程初始化过
                    if (cells == as) {
                        //初始容量为2
                        Cell[] rs = new Cell[2];
                        //当前线程会进行hash然后放置到cell数组中
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    //释放锁
                    cellsBusy = 0;
                }
                // 初始化成功直接返回
                // 因为增加的值已经同时创建到Cell中了
                if (init)
                    break;
            }

            //<4> 对应<3> 已经有线程在初始化Cell数组了 也就是<3>中的cellsBusy = 1，那么尝试cas更新base。如果成功直接返回
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```
**总结一下：**
1. 如果Cells数组未初始化，当前线程会尝试占有cellsBusy锁并创建cells数组；
2. 如果当前线程尝试创建Cells数组时，发现有其它线程已经在创建了，就尝试更新base，如果成功就返回；
3. 通过线程的probe值找到当前线程应该更新Cells数组中的哪个Cell；
4. 如果当前线程所在的Cell未初始化，就尝试占有cellsBusy锁并在相应的位置创建一个Cell；
5. 尝试CAS更新当前线程所在的Cell，如果成功就返回，如果失败说明出现冲突；
6. 当前线程更新Cell失败后并不是立即扩容，而是尝试更新probe值后再重试一次；
7. 如果在重试的时候还是更新失败，就扩容；
8. 扩容时当前线程占有cellsBusy锁，并把数组容量扩大到两倍，再迁移原cells数组中元素到新数组中；
9. cellsBusy在创建Cells数组、创建Cell、扩容Cells数组三个地方用到；
****
#### sum 方法
获取LongAdder中真正存储的值的大小，通过把base和所有段相加得到。
```c
public long sum() {
        Cell[] as = cells;
        Cell a;
        //<1> sum初始等于base
        long sum = base;
        //<2> 如果Cells不为空
        if (as != null) {
            //遍历所有的Cell
            for (int i = 0; i < as.length; ++i) {
                //如果所在的Cell不为空，就把它的value累加到sum中
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        //返回sum
        return sum;
    }
```
`note：`如果前面已经累加到sum上的Cell的value有修改，没法计算到了。所以说LongAdder是最终一致性
****
#### LongAdder VS AtomicLong
```c
public class LongAddrVSAtomicLongTest {
    public static void main(String[] args) {
        testAtomicLongVSLongAdder(1,10000000);
        testAtomicLongVSLongAdder(10,10000000);
        testAtomicLongVSLongAdder(20,10000000);
        testAtomicLongVSLongAdder(40,10000000);
        testAtomicLongVSLongAdder(80,10000000);
    }
    static void testAtomicLongVSLongAdder(final int threadCount,final int times){
        try {
            System.out.println("threadCount: " + threadCount + ",times: " + times);
            long start = System.currentTimeMillis();
            testLongAdder(threadCount,times);
            System.out.println("LongAdder elapse: " + (System.currentTimeMillis() - start) + "ms");
            long start2 = System.currentTimeMillis();
            testAtomicLong(threadCount,times);
            System.out.println("Atomic elapse: " + (System.currentTimeMillis() - start2) + "ms");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static void testLongAdder(final int threadCount,final int times) throws InterruptedException {
        LongAdder longAdder = new LongAdder();
        List<Thread> list = new ArrayList<>();
        for (int i = 0; i < threadCount; i++) {
            list.add(new Thread(()->{
                for (int j = 0; j < times; j++) {
                    longAdder.add(1);
                }
            }));
        }
        for (Thread thread : list) {
            thread.start();
        }
        for (Thread thread : list) {
            thread.join();;
        }
    }

    static void testAtomicLong(final int threadCount,final int times) throws InterruptedException {
        AtomicLong atomicLong = new AtomicLong();
        List<Thread> list = new ArrayList<>();
        for (int i = 0; i < threadCount; i++) {
            list.add((new Thread(()->{
                for (int j = 0; j < times; j++) {
                    atomicLong.incrementAndGet();
                }
            })));
        }
        for (Thread thread : list) {
            thread.start();
        }
        for (Thread thread : list) {
            thread.join();
        }
    }

}
```
#### 结果：
```c
threadCount: 1,times: 10000000
LongAdder elapse: 219ms
Atomic elapse: 72ms
threadCount: 10,times: 10000000
LongAdder elapse: 232ms
Atomic elapse: 1948ms
threadCount: 20,times: 10000000
LongAdder elapse: 462ms
Atomic elapse: 4113ms
threadCount: 40,times: 10000000
LongAdder elapse: 952ms
Atomic elapse: 9520ms
threadCount: 80,times: 10000000
LongAdder elapse: 1508ms
Atomic elapse: 21143ms
```
>可以看到线程数越多，竞争越多的情况下，对LongAdder的性能没有太大的影响而AtomicInteger则越来越慢。
****
### 总结
1. **LongAdder通过base和cells数组来存储值，无竞争的时候直接cas的修改base的值，出现竞争创建cell使用分段的思想来提交性能**
2. **不同的线程会hash到不同的cell上去更新，减少了竞争**
3. **LongAdder中Cell的数组最大容量就是当前cpu数**
4. **LongAddr 是最终一致性的。**
