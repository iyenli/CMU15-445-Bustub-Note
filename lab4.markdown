---
layout: post
title:  "CMU 15-441 Lab 4"
date:   2022-03-31 20:35:49 +0800
categories: database
tags: ["database"]
---

# About

# Lab 4

这个Lab 比起Lab 2来说还是简单一点。下面是三种事务隔离级别的效果和对应的锁策略：

- 未提交读（Read uncommitted） 指的是如果TX Abort. 他写过的数据可能被读走。【这不是无锁】需要在写的时候上写锁（互斥锁），写完放锁。读的时候不能有写锁，不能加读锁。(参考`Abort Reason`)
- 已提交读（Read committed）不会发生上述的脏读现象。但是不可重复读：读完之后被修改，再读可能读到不一致的。为了解决脏读，在Commit 之前是不可以放写锁的。然后读是不能有写锁，而且要上读锁防止被写者拿走。
- 可重复读（Repeatable read）解决了不可重复读的问题。这就需要2PL。Growing阶段拿锁，Shrinking阶段放锁。优化是在Shrinking阶段先放读锁，再放写锁。

- `upgrading`字段是因为如果有两个需要升级的线程，那么可能造成死锁；在同一时刻只能由一个线程在请求升级。

## 哈希表 Emplace

初始化LockRequestQueue的时候有问题，是因为**`mutex`和`condition_variable`不能被复制或移动**.   cpp_ref上面的原话是:

> both the copy constructor and assignment operator are deleted for this type.

So, 如果想往哈希表里面增加一个条目，应该原地构造(emplace). Map的原地构造有一定问题，因为Map需要构造Key & Value. 如果不指定，可能不知道如何分配这些参数。有个人举了个例子：

```c++
map<string, complex<double>> scp;
scp.emplace("hello", 1, 2); // 无法区分哪个参数用来构造 key 哪些用来构造 value
string s("hello", 1), complex<double> cpx(2) // scp['h'] = 2
string s("hello"), complex<double> cpx(1, 2) // scp["hello"] = 1 + 2i
```

那么只能自己来指定了。所以，如果Map采用原地构造，应该自己来解决：使用make_tuple来保证某几个参数是一组的，另外几个参数是一组的。

```c++
std::tuple<unwrap_decay_t<Types>...>(std::forward<Types>(args)...);
```

这时候是可能造成歧义的。我们看cpp_ref的一段示例程序。

```c++
int main()
{
    std::unordered_map<std::string, std::string> m;
 
    // uses pair's move constructor
    m.emplace(std::make_pair(std::string("a"), std::string("a")));
    // uses pair's converting move constructor
    m.emplace(std::make_pair("b", "abcd"));
    // uses pair's template constructor
    m.emplace("d", "ddd");
    // uses pair's piecewise constructor
    m.emplace(std::piecewise_construct,
              std::forward_as_tuple("c"),
              std::forward_as_tuple(10, 'c'));
    // as of C++17, m.try_emplace("c", 10, 'c'); can be used
    }
}
```

emplace参数可以直接被解释为pair, 可以被解释为构造函数参数，还可以被直接解释为KV（参考倒数第二个原地构造方法）。因此，需要在第一个参数加上一个空类。表示"后面两个是用来构造的，而且只能使用某种参数组合的构造函数"。

## CV

最近刚好在学原语。但是同步原语的基本理解和使用是两回事…贴一个还不错的原语介绍贴。

```c++
std::mutex mtx; // 全局互斥锁.
std::condition_variable cv; // 全局条件变量.
bool ready = false; // 全局标志位.

void do_print_id(int id)
{
    std::unique_lock <std::mutex> lck(mtx);
    while (!ready) // 如果标志位不为 true, 则等待...
        cv.wait(lck); // 当前线程被阻塞, 当全局标志位变为 true 之后,
    // 线程被唤醒, 继续往下执行打印线程编号id.
    std::cout << "thread " << id << '\n';
}

void go()
{
    std::unique_lock <std::mutex> lck(mtx);
    ready = true; // 设置全局标志位为 true.
    cv.notify_all(); // 唤醒所有线程.
}

int main()
{
    std::thread threads[10];
    // spawn 10 threads:
    for (int i = 0; i < 10; ++i)
        threads[i] = std::thread(do_print_id, i);

    std::cout << "10 threads ready to race...\n";
    go(); // go!

  for (auto & th:threads)
        th.join();

    return 0;
}
```

CV Wait 要提供锁是设计原语必须的事情。因为你要Wait就要有锁，不然可能导致危险（一睡不醒/睡前被叫醒）等。Wait进去之后会帮你释放掉，保证其他阻塞在竞争上的线程正常执行，然后一旦被唤醒，那么就能lock了。(如果返回一定是拿到Lock了，不必担心)

此外就是最好使用Predicate(谓词从句)来帮助写代码。否则你恐怕需要`唤醒 - 拿到Latch - 看一眼是否能拿到真正的Tuple锁 - While循环继续睡眠`。这样代码量会多。把While里面的代码写进Predicate即可。

>  predicate which returns false if the waiting should be continued.

## Debug 心得

