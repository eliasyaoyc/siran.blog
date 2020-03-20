---
title: "如何实现LFU算法"
date: 2020-03-08T16:37:42+08:00
draft: false
banner: "/img/blog/Java基础/WechatIMG3.jpeg"
author: "Siran"
summary: ""
tags: ["算法"]
categories: ["算法"]
keywords: ["算法","基础"]
---
设计并实现最不经常使用`（LFU）缓存的数据结构`。它应该支持以下操作：get 和 put。

get(key) - 如果键存在于缓存中，则获取键的值（总是正数），否则返回 -1。

put(key, value) - 如果键不存在，请设置或插入值。当缓存达到其容量时，它应该在插入新项目之前，使最不经常使用的项目无效。在此问题中，当存在平局（即两个或更多个键具有相同使用频率）时，最近最少使用的键将被去除。

示例：
```c
LFUCache cache = new LFUCache( 2 /* capacity (缓存容量) */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回 1
cache.put(3, 3);    // 去除 key 2
cache.get(2);       // 返回 -1 (未找到key 2)
cache.get(3);       // 返回 3
cache.put(4, 4);    // 去除 key 1
cache.get(1);       // 返回 -1 (未找到 key 1)
cache.get(3);       // 返回 3
cache.get(4);       // 返回 4
```
****
代码：
```c
 class LFUCache {
     //key和Node
     private Map<Integer, LFUNode> cache = new HashMap<>();
     //次数和node
     private Map<Integer, LinkedList<LFUNode>> frequence = new HashMap<>();
     //总容量
     private int capacity;
     //次数最小的key
     private int min;
 
     public class LFUNode {
         public int key;
         public int value;
         private int fre = 1;
 
         public LFUNode(int key, int value) {
             this.key = key;
             this.value = value;
         }
 
         public LFUNode() {
 
         }
     }
 
     public LFUCache(int capacity) {
         this.min = 0;
         this.capacity = capacity;
     }
 
     //添加节点
     public void add(LFUNode node) {
         //通过该节点的次数，去frequence获取所有相同次数的Node列表
         LinkedList<LFUNode> orDefault = frequence.getOrDefault(node.fre, new LinkedList<>());
         //把当前节点添加进去
         orDefault.add(node);
         //在放回去
         frequence.put(node.fre,orDefault);
         //缓存添加该节点
         cache.put(node.key,node);
         //使用频率最小的就是当前节点的频率。每次新增节点必定是最小的那个
         min = node.fre;
     }
 
     public int get(int key) {
         //通过key去缓存中获取节点
         LFUNode node = cache.get(key);
         //不存在直接返回-1
         if (node == null)
             return -1;
         //存在，那么把该节点的使用频率加+1，在塞入frequence中
         else {
             int fre = node.fre;
             LinkedList<LFUNode> lfuNodes = frequence.get(fre);
             lfuNodes.remove(node); 
             //这里的判断 如果当前获取的节点的频率就是最小的，并且只有唯一一个，那最小频率+1.
             if (fre == min && lfuNodes.size() == 0)
                 min = fre + 1;
             node.fre = node.fre + 1;
             LinkedList<LFUNode> lfuNodes1 = frequence.getOrDefault(node.fre, new LinkedList<>());
             lfuNodes1.add(node);
             frequence.put(node.fre,lfuNodes1);
             return node.value;
         }
     }
     
     //put操作结合了add和get
     public void put(int key, int value) {
         if(capacity == 0)
            return;
         LFUNode node = cache.get(key);
         if (node == null) {
             if (cache.size() >= capacity) {
                 LinkedList<LFUNode> lfuNodes = frequence.get(min);
                 LFUNode remove = lfuNodes.removeFirst();
                 cache.remove(remove.key);
             }
             add(new LFUNode(key, value));
         } else {
             node.value = value;
             int fre = node.fre;
             LinkedList<LFUNode> lfuNodes = frequence.get(fre);
             lfuNodes.remove(node);
             if (fre == min && lfuNodes.size() == 0)
                 min = fre + 1;
             node.fre = node.fre + 1;
             LinkedList<LFUNode> orDefault = frequence.getOrDefault(node.fre, new LinkedList<>());
             orDefault.add(node);
             frequence.put(node.fre, orDefault);
         }
     }
 }
```
