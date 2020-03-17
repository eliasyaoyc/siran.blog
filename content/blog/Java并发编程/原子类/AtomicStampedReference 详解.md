---
title: "AtomicStampedReference 详解"
date: 2020-03-05T12:37:42+08:00
draft: false
banner: "/img/blog/banners/006tKfTcly1g0n3qqw0xqj31400u0hdt.jpg"
author: "Siran"
summary: "AtomicStampedReference是jdk1.5出的一个类，用于解决其他原子类无法解决的ABA问题。"
tags: ["原子类"]
categories: ["并发编程"]
keywords: ["原子类","基础"]
---
### 问题
1. 什么是ABA?
2. ABA会出现什么问题？
3. ABA如何解决？
4. AtomicStampedReference是怎么解决ABA的？
****
### 简介
>AtomicStampedReference是jdk1.5出的一个类，用于解决其他原子类无法解决的ABA问题。
****
### ABA 问题
>首先在多线程的case中，我们要正确的修改一个变量的值，要么使用锁让多线程排队依次执行。但是效率会变低，稍微择中的办法就是自旋，使用CAS来修改变量的值，会不断的自旋来修改直到成功为止。
那么当某线程连续读取同一个内存地址两次，两次得到的值是一样的，它就认为没有被修改，然后进行修改。但是，可能出现另一个线程在这之前先把值修改成了B然后又修改成了A。这时还简单地认为“没有修改过”显然是错误的。

**比如，两个线程按下面的顺序执行：**

（1）线程1读取内存位置X的值为A；

（2）线程1阻塞了；

（3）线程2读取内存位置X的值为A；

（4）线程2修改内存位置X的值为B；

（5）线程2修改又内存位置X的值为A；

（6）线程1恢复，继续执行，比较发现还是A把内存位置X的值设置为C；
![](/img/blog/原子类/ABA.png)
****
### ABA 代码实操
模拟一个无锁的栈，出栈和入栈。看一下ABA的问题
```c
static class Stack{
        private AtomicReference<Node> top = new AtomicReference<>();
        static class Node{
            int val;
            Node next;
            public Node(int val) {
                this.val = val;
            }
        }

        public Node pop(){
            for (;;){
                Node node = top.get();
                if (node == null)
                    return null;
                Node next = node.next;
                if (top.compareAndSet(node,next)){
                    node.next = null;
                    return node;
                }
            }
        }
        public void push(Node node){
            for (;;){
                Node node1 = top.get();
                node.next = node1;
                if(top.compareAndSet(node1,node))
                    return;
            }
        }
    }
//测试方法
public static void main(String[] args) throws InterruptedException {
        Stack aba = new Stack();
        aba.push(new Stack.Node(3));
        aba.push(new Stack.Node(2));
        aba.push(new Stack.Node(1));
        Thread thread1 = new Thread(() -> {
            aba.pop();
        }, "线程1");
        Thread thread2 = new Thread(()->{
            Stack.Node A = aba.pop();
            Stack.Node B = aba.pop();
            aba.push(A);
        },"线程2");
        thread1.start();
        thread2.start();
    }
```
假如，我们初始化栈结构为 top->1->2->3，然后有两个线程分别做如下操作：

（1）线程1执行pop()出栈操作，但是执行到 if(top.compareAndSet(t,next)){这行之前暂停了，所以此时节点1并未出栈；

（2）线程2执行pop()出栈操作弹出节点1，此时栈变为 top->2->3；

（3）线程2执行pop()出栈操作弹出节点2，此时栈变为 top->3；

（4）线程2执行push()入栈操作添加节点1，此时栈变为 top->1->3；

（5）线程1恢复执行，比较节点1的引用并没有改变，执行CAS成功，此时栈变为 top->2；

为什么 top->2 ？不是应该变成 top->3 吗？

那是因为线程1在第一步保存的next是节点2，所以它执行CAS成功后top节点就指向了节点2了。

下面具体断点的过程：
>首先释放线程2，让它先执行

![](/img/blog/原子类/线程2.png)
![](/img/blog/原子类/线程2值.png)

>线程2执行完后的栈中的值为1 -> 3，因为线程1一开始保存的next值是2，但是它cas操作的时候发现第一个元素还是1，就觉得没有被修改，那么就把线程1中的next值放到栈顶

![](/img/blog/原子类/线程1.png)
****
### 源码分析
>AtomicStampedReference是通过版本号来解决ABA问题的
#### 内部类    
```c
private static class Pair<T> {
        final T reference;
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }
```
内部类Pair，将元素值和版本号绑定在一起。
****
#### 主要属性
```c
    private volatile Pair<V> pair;
```
****
#### 构造方法
```c
//构造方法需要传入初始值及初始版本号
public AtomicStampedReference(V initialRef, int initialStamp) {
        pair = Pair.of(initialRef, initialStamp);
    }
```
****
#### compareAndSet 方法
```c
public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        //获取当前的（元素值，版本号）对
        Pair<V> current = pair;
        return
            //判断引用是否发生变化
            expectedReference == current.reference &&
            //判断版本号是否发生变化
            expectedStamp == current.stamp &&
            //引用赋值
            ((newReference == current.reference &&
              //版本号赋值
              newStamp == current.stamp) ||
             //cas的构造新的Pair对象和修改值
             casPair(current, Pair.of(newReference, newStamp)));
    }
```
1. `如果元素值和版本号都没有变化，并且和新的也相同，返回true；`
2. `如果元素值和版本号都没有变化，并且和新的不完全相同，就构造一个新的Pair对象并执行CAS更新pair。`

可以看到，java中的实现跟我们上面讲的ABA的解决方法是一致的。

首先，使用版本号控制；

其次，不重复使用节点（Pair）的引用，每次都新建一个新的Pair来作为CAS比较的对象，而不是复用旧的；

最后，外部传入元素值及版本号，而不是节点（Pair）的引用。
****
### 总结
1. **在多线程环境下使用无锁结构要注意ABA问题**
2. **ABA的解决一般使用版本号来控制，并保证数据结构使用元素值来传递，且每次添加元素都新建节点承载元素值**
3. **AtomicStampedReference内部使用Pair来存储元素值及其版本号**
4. **AtomicMarkableReference，它不是维护一个版本号，而是维护一个boolean类型的标记，标记值是否有修改**
