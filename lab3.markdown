---
layout: post
title:  "CMU 15-441 Lab 3"
date:   2022-03-28 20:35:49 +0800
categories: database
tags: ["database"]
---

# About

- 对于CMU的学生来说，每个Lab都有一个月的时间。但我希望尽快搞定，所以从Lab 3开始参考了一个仅有几行关键代码示例的通关博客。仅供参考。[CMU15-445 数据库实验全满分通过笔记](https://blog.csdn.net/twentyonepilots/article/details/120868216). 选中的原因是因为他前面两个lab提到的一些关键点和我的体会很相似。所以后面这篇博客提到的我应该不会再提及。

# Lab 3

在这个Lab中，需要实现火山模型。主要工作量在于读懂数量庞大的工具类。分散在`storage`(Value, tuple, page), `executors`(plan, exec),  `catalog`(meta) 内。包括类型，值，Tuple, TablePage, 迭代器, RID(Page + Slot number唯一确定一个Entry)等。本身Lab难度略难于Lab 1, 主要是体会帮你设计的工具类，并且学会使用它。

- Table & Table heap, Table继承自Page, Table heap是Table的容器，有迭代器
- Update, Insert 的时候**不可以产生元组结果**，否则不符合语义。只需要While直到子节点不返回，然后**返回False**即可, 查看`engine`理解结果的产生机制。
- 如果你数据库本身应用也不牢固（比如我），那你在这个Lab可能遇到(没什么人遇到过的)困难。学习一些主要的概念：
  - Group By, Aggreagate, Join, Distinct

- Lab 3的测试用例多得令人感动… 这降低了他的难度

## Debug心得

- 有的节点说明了“只在根节点出现“，那么汇总好结果/执行完之后返回False(不读取结果集)；

- 使用`Evaluate`接口而不是自己去拼装值；因为执行Query时可能Schema发生改变，自己拼装Column的值可能无效导致异常；比如你想获得某列的值: `plan_->OutputSchema()->GetColumn(i).GetExpr()->Evaluate(&tuple, &(table_info_->schema_))`

- 上面的Bug其实都不难，最让我觉得XXX的Bug是…我写反了下面两个东西。

  ```c++
  left_not_end_ = left_executor_->Next(&left_tuple_, &left_rid_);
  right_not_end_ = right_executor_->Next(&right_tuple_, &right_rid_);
  // left, right的参数和executor写反了
  ```

  然后报的错也不是正确性错误，居然是内存泄漏。

  ```
  ==2473== Invalid read of size 4
  函数调用栈
  ==2473==  Address 0x836e2ac is 0 bytes after a block of size 12 alloc'd
  函数调用栈
  ```

  不过也算趁机了解了一下Valgrind的内存泄漏判断机制以及如何根据报错解决。但是当我真的定位到这个Bug之后，确实差点把键盘砸了…

最后成绩勉强在非CMU学生前50…

<img src="https://s2.loli.net/2022/03/29/M1RdIrAnTHPVj4O.png" alt="image-20220329202221254" style="zoom: 67%;" />
