---
title: "如何实现LRU算法"
date: 2020-03-08T15:37:42+08:00
draft: false
banner: "/img/blog/Java基础/WechatIMG4.jpeg"
author: "Siran"
summary: ""
tags: ["算法"]
categories: ["算法"]
keywords: ["算法","基础"]
---

LRU 缓存淘汰算法就是一种常用策略。`LRU 的全称是 Least Recently Used`，也就是说我们认为最近使用过的数据应该是是「有用的」，很久都没用过的数据应该是无用的，内存满了就优先删那些很久没用过的数据。

思路：每次操作都把`操作的值`放在第一位，如果添加元素超过了指定的容量，那就把末尾的淘汰(最旧的)。

```c
public class LRUCache {
    private Hashtable<Integer, LinkNode> lru = new Hashtable<>();
    //容量
    private int capacity;
    //当前的节点个数
    private int count;
    //两个哑巴节点 方便操作
    private LinkNode head;
    private LinkNode tail;

    class LinkNode {
        int key;
        int value;
        LinkNode pre;
        LinkNode next;

        public LinkNode(int key, int value) {
            this.key = key;
            this.value = value;
        }

        public LinkNode() {

        }
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.count = 0;

        head = new LinkNode();
        tail = new LinkNode();
        head.next = tail;
        tail.pre = head;
    }

    //移动到队列的第一位
    private void moveToHead(LinkNode node) {
        removeNode(node);
        addNode(node);
    }

    //获取最后一个节点
    private LinkNode getLast() {
        LinkNode last = tail.pre;
        removeNode(last);
        return last;
    }

    //删除节点
    private void removeNode(LinkNode node) {
        LinkNode pre = node.pre;
        LinkNode next = node.next;
        pre.next = next;
        next.pre = pre;
    }
 
    //删除节点  
    private void addNode(LinkNode node) {
        node.pre = head;
        node.next = head.next;
        head.next.pre = node;
        head.next = node;
    }

   
    public int get(int key) {
        if (lru.get(key) != null) {
            LinkNode node = lru.get(key);
            moveToHead(node);
            return node.value;
        }
        return -1;
    }

    public void put(int key, int value) {
        LinkNode node = lru.get(key);
        if (node == null) {
            LinkNode n = new LinkNode(key, value);
            addNode(n);
            count++;
            lru.put(key, n);

            if (count > capacity) {
                LinkNode last = getLast();
                removeNode(last);
                lru.remove(last.key);
            }
        } else {
            node.value = value;
            moveToHead(node);
        }
    }
}

```