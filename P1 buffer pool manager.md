
### LRU-K Replacer

LRU Replacer 是管理缓存的工具，你希望在缓存已满又需要分配新的缓存时，合理的决定驱逐哪个，防止被驱逐的 node 短时间内再被访问。LRU 最近最少访问，会维护一个链表和一个 unordered_map, 每次 touch 一个数据，如果在链表里就拿出来放在队尾，否则加入到队尾。然后 evict 就赶走队头的，这样队头的永远是最远被使用的节点。

LRU-K Replacer 将队列分为了两个，一个是 touch 次数小于 K 的 (`under_k_list`)，一个是 touch 次大于 K 的 (`k_list`)。记录最多 K 条访问 Time Stamp. 这是为了避免大量只读一次的数据驱逐掉几乎整个 Cache. （比如 Scan 操作） 一般 K 不会很大 (<10)，避免 LRU-K List 中的数据几乎不出来。当需要 Evict 的时候，先驱逐 under_k_list 的最早进入的节点（相当于都没有累计到 K 次访问，这小子花的时间最长），如果没有可驱逐的，那么驱逐 `under_k_list` 倒数第 K 次访问时间最早的 Frame. （都是 K 次访问，这小子占据的时间最久）

和 LRU 不同，LRU-K 对每个节点要记录 K 个时间。时间戳用链表存储，返回加入这个时间戳后是否需要转移到 k_list 中，这可以避免还要 caller 确定调用前 list 多长调用后多长：

```c++
auto AppendHistory(size_t timestamp) -> bool {
if (history_.size() == k_) {
  history_.pop_back();
}
history_.emplace_front(timestamp);
if (!in_k_list_ && history_.size() == k_) {
  in_k_list_ = true;
  return true;
}
return false;
}
```

此外，每个 LRUKNode 都还要维护这个节点对应的是哪个 frame, K 是多少，是否可驱逐。注意，buffer pool 应该提供 pin 住缓存的接口，避免使用中的缓存页被驱逐。

这个 Macros 是经常见到的惯用法，用于删除 class 的 copy / move 函数。

```C++
#define DISALLOW_COPY(cname)                                    \
  cname(const cname &) = delete;                   /* NOLINT */ \
  auto operator=(const cname &)->cname & = delete; /* NOLINT */
#define DISALLOW_MOVE(cname)                               \
  cname(cname &&) = delete;                   /* NOLINT */ \
  auto operator=(cname &&)->cname & = delete; /* NOLINT */
```

我们希望能靠 `frame_id` O (1) 的找到 Node，这需要一个哈希表。但是我们又希望 Under K List 保持顺序（方便移除最早来的 Frame），因此需要哈希表持有一个指向 List 的迭代器，**注意，移除 List 元素，只会导致被移除的迭代器失效，这是 List Container 的优势。** 我们保证 `under_k_list` 是有序的，最早加入的一定在队列最前面。这样删除的时候只需要 O (1) 就可以了。

对于 K List，Trade Off 是随机访问和迭代器不失效的取舍。因为 K-Distance 的变化后，并不像 Under-K List, K List 需要把他插入到数据结构中间，这难免需要二分查找，但是二分查找需要随机访问。也就是说，哈希表需要维护 `frame_id` 和随机访问迭代器的映射，但是移除随机访问迭代器，以 vector 为例，后面所有迭代器也都会失效。有两种方案：

- 每次 Evict 的时候，Lazy 的遍历一次 K List，O (n)
- 每次有新的 K- Distance 被计算出来的时候，Locate 到对应位置, Record Access O (n), Evict O (1)

我认为第一种更好。首先显然 Record 是要更多的，其次从实现上只需要 `unordered_map` 就不需要 `std::list` 就可以 Cover 了。（但我还是习惯叫 `k_list`）

注意 k_list 和 under_k_list 都需要一把 latch. 注意维护 `curr_size`, 这说的是这个 buffer pool 还有多少没有被 pin 住可以驱逐的节点。在节点被驱逐，被加入，被 pin/unpin 的时候都需要维护。

**Evict**: 先看 under_k_list 从前往后，遇到的第一个可以驱逐的就驱逐；如果没有，遍历一次 k_list, 找到 K-Distance 最长的删除

**RecordAccess**: 如果在 under_k_list, AppendHistory 之后如果发现进入 k_list, 那么删除后进入 k_list. 如果在 k_list, 只记录时间戳，如果不在现有的 list 中，那么需要加入到 under_k_list / k_list. (取决于 K 是否为 1)。注意，最后一种情况不需要调用 Evict.

时间戳建议使用 `std::atomic`: `auto ts = current_timestamp_.fetch_add(1);` .
### Disk Scheduler

把 `disk_manager_->ReadPage` 进行包装。使用的是 promise-future 机制，其实就是一个通过堆上内存进行通信信道的两个 port. 比如通过 `std::packaged_task` 创建一个闭包任务的时候，可以获取 future, 「假装」结果已经拿在手里了，只有当 future 真的被获取值，才会等待异步任务完成真的返回值。这样可以最大的提高并行度。

在提交任务时，我们先创建 promise（发送结果的 port），然后获取 promise 对应的 future, 然后将 Disk 任务发送到 `Channel<std::optional<DiskRequest>> request_queue_;` 中。

```C++
auto WritebackPage(frame_id_t frame_id) -> bool {
	auto promise = disk_scheduler_->CreatePromise();
	auto future = promise.get_future();
	disk_scheduler_->Schedule({true, pages_[frame_id].GetData(),  pages_[frame_id].GetPageId(), std::move(promise)});
	return future.get();
}
```

这里返回的时候调用了 get (), 其实使用的是同步写的方法。这里的 Channel 是 Disk Scheduler 的核心，本质上是一个队列加上 CV 和锁的实现：

```C++
template <class T>
class Channel {
 public:
  void Put(T element) {
    std::unique_lock<std::mutex> lk(m_);
    q_.push(std::move(element));
    lk.unlock();
    cv_.notify_all();
  }

  auto Get() -> T {
    std::unique_lock<std::mutex> lk(m_);
	// 1. 进入wait之前会检查lambda，如果为true那么会拿着锁进入下面的操作
	// 2. 如果返回false, 那么会放掉锁，阻塞当前线程
	// 3. 当被notify之后，会把锁拿着，判断lambda是否为真，如果拿不到锁或者lambda、
	// 为false，那么继续阻塞，否则拿着锁走接下来的逻辑
    cv_.wait(lk, [&]() { return !q_.empty(); });
    T element = std::move(q_.front());
    q_.pop();
    return element;
  }

 private:
  std::mutex m_;
  std::condition_variable cv_;
  std::queue<T> q_;
};
```
### Buffer Pool Manager

接下来在上面的基础做 BPM, 首先是性能建议：使用大锁。根据 8-2 法则，80%的代码决定了 20%的性能，使用细粒度的锁看上去很高效，但是 debug 复杂不说，还没有肉眼可见的性能收益。因为内存操作其实是非常快的，这个实验的瓶颈在：

1. LRU Replacer: 通过 perf 工具发现了 LRU Replacer 消耗了大量的时间
2. 为随机读写增加了 1ms 的时间，为顺序读写增加了 0.1ms 的时间

接下来先快速介绍一下如何正确实现 BPM. 在构造函数要 `new Page[pool_size]` , 然后把所有的 frame_id 都加入到 free_list 中。

注意，这里的 page_id 可以理解为磁盘中的 4K, 而 frame_id 则是 BPM 连续内存中的 id. 这里模拟 `AllocatePage` 分配磁盘页的方式来获得 page_id. 注意，frame_id 完全是 BPM 内部的索引，给用户看到的是一个带 cache 的读写磁盘 page 的抽象，并且这个抽象还能正确解决 dirty, 写回，cache 管理等。

**NewPage**: 检查是否有 free_list 可用，如果没有那么 `replacer.evict` 一个 frame, 随后 `RecordAccess` 并将这个 frame 设置为不可被驱逐，分配一个 page_id, 然后记录 frame 对应的的 page_id 并且将 pin_count 设置为 1.

**FetchPage**: 先通过 frame_id 找到对应的 frame, 如果找得到，那么只要增加 pin_count, 设置 `evictable` 为 false 并且 RecordAccess. 如果找不到，那需要像上面一样分配一个 new page 并且重新设置 page_table, 将磁盘中的 page 读出来，然后仍然需要 `RecordAccess` 并且 `SetUnevictable`.

**UnpinPage**: 只有在 frame_id 存在并且 `pin_count` 不为 0 的时候才可以 `unpin`, 只需要 `--pin_count`, 如果为 0，`再SetEvictable` . 注意，在这个过程中要正确设置 page 元数据 is_dirty.

**FlushPage**: 写回 Page 并且设置 is_dirty 为 false. DeletePage 则是先要写回（在 pin_count 为 0 的前提下），然后再驱逐 Page, 并且归还给磁盘 (`DeallocatePage`), 从 LRU 中移除这个 frame 并且从 page_table 等删掉，然后把 frame_id 重新加入 free_list.

