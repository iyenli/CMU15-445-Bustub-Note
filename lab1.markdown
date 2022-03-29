---
layout: post
title:  "CMU 15-441 - Lab 1"
date:   2022-03-24 20:35:49 +0800
categories: database
tags: ["database"]
---

# About

CMU15-445想必不必介绍。由于教师Andy Pavlo要求不能将实现开源，因此本Repo只包含写的笔记。在正式开始写这个Lab之前，你需要：

- 注册Grade scope账号，因为本身的测试用例之弱小会让这个Lab的意义大打折扣
- 去听15-445第一节课，里面有课堂DJ的开场曲（

# Lab 1

## LRU_REPLACER

https://en.cppreference.com/w/cpp/thread/lock_guard

函数总有很多出口，难道每次都要Lock & Unlock嘛？这个接口提供了一种包装，只要控制流出作用域，那么就自动放锁。

LRU如何在O(1)完成更新，获取，这个是有例子的(链表+哈希表)。但是如何用Relative Mordern C++重写一次是值得自己试试的。在Lab 1中的线上评分，由于数据量庞大，自己写的链表可能出现超时的问题。因此，可以用list + 迭代器完成同样的功能。好处除了性能的提高之外，还有我们不必自己写结构体，不必小心的处理内存泄漏问题。如果想自己试试，可以了解：

- 如何表示空迭代器（提示：默认构造函数）
- 如何保证迭代器不失效（提示：cpp reference 有失效条件）
- 迭代器会随着push/pop改变指向的元素吗？（当然不会）

最后请注意使用合适的数据结构即可。比如能用vector的(frame_id是连续自然数)那就没有必要用哈希表了。事实上，这里的性能差距是大的。

## Buffer Pool Manager

多线程编程有一个特点，大部分的公共数据结构都需要Latch来读写。但是不可避免地出现一些代码重复的片段。那么就需要这样一种函数（学名还不清楚）：进去需要拿着锁进去（调用的地方必须在锁内），函数本身不影响锁。这样的函数就可作为工具函数被复用了。

## Summary

总的来说，Lab 1还是非常简单易上手的。花费的大部分时间都是在阅读接口，思考关系。参考资料：一些LRU的实现方法。这一个Lab 一般来说不需要观看课程。最后，贴一个个人认为本Lab最难发现的一个Bug, 仅在Scope能测出来：`UnpinPgImp`中只有`is_dirty_`为真时设置页面为脏，如果`is_dirty_`为假，页面Dirty Bit不能被false覆盖！此外，Grade scope的脚本也有小Bug，他会要求`unpin`之后的`is_dirty_`和给定的保持一致。但是没有考虑`unpin`之后为0马上flush的场景。

<img src="https://s2.loli.net/2022/03/14/mKhLPR7gvAHCEQa.png" alt="image-20220314094007439" style="zoom:67%;" />

最后似乎只有Non CMU的70名左右…
