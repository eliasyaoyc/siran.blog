---
title: "StringBuilder 解析"
date: 2020-03-08T16:37:42+08:00
draft: false
banner: "/img/blog/banners/006tNc79ly1g1wrwnznblj31400u0x6p.jpg"
author: "Siran"
summary: ""
tags: ["Java"]
categories: ["Java基础"]
keywords: ["Java","基础"]
---
#### 题目
给定两个字符串 s1 和 s2 ,计算出将s1 转换为 s2 所使用的最小操作数。

你可以对一个字符串进行如下三种操作：
1. 插入一个字符串
2. 删除一个字符串
3. 替换一个字符

#### 示例1：
>输入: s1 = "horse" , s2 = "ros"
>
>输出：3
>
>解释:
>
>horse -> rorse(将 h 替换为 r)
>
>rorse -> rose(删除 r)
>
>rose -> ros(删除 e)

#### 示例2：
>输入: s1 = "intention" , s2 = "execution"
>
>输出：5
>
>解释:
>
>intention -> inention(删除 t)
>
>inention -> enention(将 i 替换为 e)
>
>enention -> exention(将 n 替换为 x)
>
>exention -> exection(将 n 替换为 c)
>
>exection -> execution(插入 u)
