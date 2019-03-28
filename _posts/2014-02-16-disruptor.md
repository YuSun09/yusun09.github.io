---
layout: post
title: "disruptor"
date: 2014-02-16
categories: opinion
tags: [resources, jekyll]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

https://juejin.im/post/5b5f10d65188251ad06b78e3

Disruptor

队列为什么要加锁？
在jdk中的队列都实现了java.util.Queue接口，在队列中又分为两类，一类是线程不安全的，ArrayDeque，LinkedList等等，还有一类都在java.util.concurrent包下属于线程安全，而在我们真实的环境中，我们的机器都是属于多线程，当多线程对同一个队列进行排队操作的时候，如果使用线程不安全会出现，覆盖数据，数据丢失等无法预测的事情，所以我们这个时候只能选择线程安全的队列。


队列是我们非常常用的数据结构，用来提供数据的写入和读取功能，而且通常在不同线程之间作为数据通信的桥梁。不过在将无锁队列的算法之前，需要先了解一下CAS（compare and swap）的原理。由于多个线程同时操作同一个数据，其中肯定是存在竞争的，那么如何能够针对同一个数据进行操作，而且又不用加锁呢？ 这个就需要从底层，CPU层面支持原子修改操作，比如在X86的计算机平台，提供了XCHG指令，能够原子的交互数值。