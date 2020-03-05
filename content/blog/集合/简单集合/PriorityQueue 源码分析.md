---
title: "PriorityQueue 源码分析"
date: 2020-02-12T19:12:42+08:00
draft: false
banner: "/img/blog/banners/00704eQkgy1fs3o6ljkknj30rs0ku4qp.jpg"
author: "Siran"
summary: "PriorityQueue里的每个元素都会进行排序，每次弹出一个元素要么是最大的要么是最小的，取决于排序规则。"
tags: ["集合"]
categories: ["集合"]
keywords: ["集合","Jdk源码","基础"]
---
## 问题
1. 什么是优先队列？
2. PriorityQueue线程安全吗？
3. PriorityQueue是有序的吗？
4. PriorityQueue如何实现？
****
## 简介
>PriorityQueue是基于Heap来实现的。
PriorityQueue里的每个元素都会进行排序，每次弹出一个元素要么是最大的要么是最小的，取决于排序规则
PriorityQueue是一个无界的队列，但是内部会有一个容量默认为11。
PriorityQueue确保子节点一定比其父节点大
假设现在往PriorityQueue中插入1,3,4,6,5,1,2

![](/img/blog/集合/priorityQeueu/WechatIMG17.png)
****
## 源码分析

#### 主要参数
```c
//默认容量
private static final int DEFAULT_INITIAL_CAPACITY = 11;
//底层使用数组来存储数据
transient Object[] queue  // non-private to simplify nested class access
//元素个数
private int size = 0;
//比较器
private final Comparator<? super E> comparator;
//修改次数
transient int modCount = 0; // non-private to simplify nested class access
```
****
#### 构造器
```c
//这些构造器可以指定容量和元素的Comparator(不指定就默认元素自身的排序)，也可以传入Collection 或者 PriorityQueue
public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }
public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }
public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }
public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // 容量校验
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }

//传入Collection
public PriorityQueue(Collection<? extends E> c) {
        //<1>.如果这个集合是SortedSet 那么赋值Comparator，调用#initElementsFromCollection()方法初始化
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        //<2>.如果就是PriorityQeueue 那么赋值Comparator，调用#initFromPriorityQueue()方法初始化
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        //<3>.如果是其他集合那么就调用#initFromCollection()方法初始化
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }

//传入PriorityQueue 同上构造器中的<2>
public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }

private void initElementsFromCollection(Collection<? extends E> c) {
        //<1> 转成数组
        Object[] a = c.toArray();
        // If c.toArray incorrectly doesn't return Object[], copy it.
        //<2> 检查
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, a.length, Object[].class);
        int len = a.length;
        if (len == 1 || this.comparator != null)
            for (int i = 0; i < len; i++)
                if (a[i] == null)
                    throw new NullPointerException();
        //<3> 赋值
        this.queue = a;
        this.size = a.length;
    }

private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
        //<1> 判断类型 如果就是PriorityQueue类型那么说明已经堆化了直接赋值
        if (c.getClass() == PriorityQueue.class) {
            this.queue = c.toArray();
            this.size = c.size();
        } else {
            //<2> 此方法会转成数组并且进行堆化
            initFromCollection(c);
        }
    }

private void initFromCollection(Collection<? extends E> c) {
        //转换为数组
        initElementsFromCollection(c);
        //堆化
        heapify();
    }

//进行堆化(二叉堆) 必须要求子节点比父节点大
private void heapify() {
        //siftDown方法在下面的删除中分析
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
```
****
#### 入队
```c
//添加一个元素，最终是调用#offer()方法
public boolean add(E e) {
        return offer(e);
    }

//这里会判断是否需要扩容，以及会进行自下而上堆化
public boolean offer(E e) {
        //<1> 为null 抛出空指针
        if (e == null)
            throw new NullPointerException();
        modCount++;
        //取元素个数
        int i = size;
        //<2> 元素的个数达到最大容量了则调用#grow()方法进行扩容
        if (i >= queue.length)
            grow(i + 1);
        //元素个数+1
        size = i + 1;
        //<3> 如果第一次添加元素，那么不用进行堆化，直接赋值
        if (i == 0)
            queue[0] = e;
        else
        //<4> 自下而上的进行堆化
            siftUp(i, e);
        return true;
    }

//扩容
private void grow(int minCapacity) {
        int oldCapacity = queue.length;
        // Double size if small; else grow by 50%
        //<1> 如果旧容量没有超过64则+2，反之加一半
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
        // overflow-conscious code
        //<2> 检查是否溢出
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        //<3> 创建出一个新容量大小的新数组并把旧数组元素拷贝过去
        queue = Arrays.copyOf(queue, newCapacity);
    }

private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

//自下而上堆化
private void siftUp(int k, E x) {
        //<1> 如果指定了比较器则调用#siftUpUsingComparator()方法
        if (comparator != null)
            siftUpUsingComparator(k, x);
        else
        //<2> 反之调用#siftUpComparable()方法，这里就分析此方法。原理都是一样的
            siftUpComparable(k, x);
    }

private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            //<1> 查找父节点的index
            int parent = (k - 1) >>> 1;
            //<2> 获得父节点的值
            Object e = queue[parent];
            //<3> 添加的值如果 >= 父节点的值，那么满足二叉堆的特性,退出循环
            if (key.compareTo((E) e) >= 0)
                break;
            //<4> 小于父节点的值，那么父节点的值和要插入的值互换位置，
            //依次类推自下而上和父节点进行比较，小就互换位置。
            //所以如果不指定排序规则那么每次取出来的值都是最小的值
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
```
还是以1,3,4,6,5,1,2 为例子一开始插入1是在4后面的如图所示，但是在进行堆化的时候发现1< 4，互换位置，1的父节点是1 不小于，满足二叉堆的特性。

![](/img/blog/集合/priorityQeueu/1581490896271.jpg)

****

#### 出队
```c
//弹出头节点，最小值或者最大值并且自上而下的进行堆化
public E poll() {
        //<1> 没有元素 返回null
        if (size == 0)
            return null;
        //<2> 元素个数-1
        int s = --size;
        modCount++;
        //<3> 获得第一个元素
        E result = (E) queue[0];
        //<4> 获取最后一个元素
        E x = (E) queue[s];
        //<5> 把最后一个元素置为null
        queue[s] = null;
        //<6> 自上而下的进行堆化
        if (s != 0)
            siftDown(0, x);
        //<7> 返回第一个元素
        return result;
    }

//和向上堆化一样这里只分析#siftDownComparable()
private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }

private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        //<1> 只需要比较一半就行了，因为叶子节点占了一半的元素
        int half = size >>> 1;        // loop while a non-leaf
        while (k < half) {
            //左节点的index
            int child = (k << 1) + 1; // assume left child is least
            //左节点的值
            Object c = queue[child];
            //右节点的index
            int right = child + 1;
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)  
                //<2> 比较左右节点取小值
                c = queue[child = right];
            //<3> 如果最后一个元素的值比当前最小的子节点都小，则退出循环
            if (key.compareTo((E) c) <= 0)
                break;
            //<4> 如果比当前最小的子节点大，则交换位置
            queue[k] = c;
            //<5> 指针移到最小子节点位置，继续往下比较
            k = child;
        }
        //<6> 找到正确的位置，放入元素
        queue[k] = key;
    }
```
**继续以1,3,4,6,5,1,2为例子**

![](/img/blog/集合/priorityQeueu/WechatIMG16.png)

**现在调用#poll()方法弹出第一个元素,最后一个元素被置为null(这里置为null的意思并不是删除。而是要移动)**

![](/img/blog/集合/priorityQeueu/WechatIMG12.png)

**因为一个节点下面有两个子节点所以只要比较一半的元素就可以。
以第一个节点开始，获取该节点的左右子节点取小值则为(1)，判断如果最后一个元素(2)比当前的最小值(1)还要小直接退出这个循环,并把最后一个元素(2)放到第一个元素的位置上**

![](/img/blog/集合/priorityQeueu/WechatIMG13.png)

**如果大于当前的最小值(1)，则交换位置继续往下比较**

![](/img/blog/集合/priorityQeueu/WechatIMG14.png)

**最终**

![](/img/blog/集合/priorityQeueu/WechatIMG15.png)

****
#### 删除
```c
//删除给定的元素
public boolean remove(Object o) {
        //<1> 获取元素的下标
        int i = indexOf(o);
        //<2> 不存在返回false
        if (i == -1)
            return false;
        else {
        //<3> 反之调用#removeAt()方法进行删除
            removeAt(i);
            return true;
        }
    }

//通过给定的元素查找元素的下标 时间复杂度为On
private int indexOf(Object o) {
        if (o != null) {
            for (int i = 0; i < size; i++)
                if (o.equals(queue[i]))
                    return i;
        }
        return -1;
    }

private E removeAt(int i) {
        // assert i >= 0 && i < size;
        modCount++;
        int s = --size;
        //<1> 如果是给定的元素是最后一个 直接删除，不用堆化
        if (s == i) // removed last element
            queue[i] = null;
        else {
            //<2> 获得要删除的值
            E moved = (E) queue[s];
            //<3> 最后元素置为null
            queue[s] = null;
            //<4> 向下堆化
            siftDown(i, moved);
            //<5> 此时给定删除元素的位置就是最后一个元素
            if (queue[i] == moved) {
                //<6> 向下堆化
                siftUp(i, moved);
                if (queue[i] != moved)
                    return moved;
            }
        }
        return null;
    }
```
***
#### 获取首元素
```c
//获取头节点 最小值或者是最大值
public E peek() {
        return (size == 0) ? null : (E) queue[0];
    }
```
****
## 总结

* **PriorityQueue不是线程安全的，任何操作都没有使用同步机制。多线程情况下可以使用PriorityBlockingQueue**
* **PriorityQueue是无序的，只有第一个元素要么是最小的要么是最大的**
* **PriorityQueue入队出对对应堆的插入元素和删除元素，**
* **PriorityQueue的入队出队的时间复杂度相比于数组而言慢为O(log(n))**
