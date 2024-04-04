这是全新的 MVCC Lab, 核心在于读操作是无锁的。
Motivation 是「我们为什么需要读最新的数据？只要读的数据是一致的就可以了。」比如 OLAP 的时候，我们在 23 点开始分析购物网站的流水，只要账对的上，我们可以获得一个 23 点的 snapshot 算流水，而不需要上大量的锁得到最新的流水。
MVCC 的基本逻辑是，事务开始的时候记录一个时间戳 (`read_ts`), 所有的读只能读到 `read_ts` 之前的版本，相当于在开始的时候做了一个 snapshot. 在 commit 的时候，如果发现写集在 `read_ts` 后被修改了就 Abort. 否则就实际写入然后将时间戳更新为 `commit_ts` .
`read_ts`, `commit_ts` 共享一套全局时间戳，每 commit 一个 `txn` 就自增一次。
### Overview
![[assets/Untitled 6.png]]
实现之前，先基于上图介绍一下基本的数据结构。
- **Heap**是**Table**的磁盘储存格式，包括若干个**Page**, **Tuple**是关系型数据库的**一行**，而**RID**是找到这一行的索引，包括**Page ID 和 Slot ID**.
一个**Page**里面包含多个**Slot**, 如果要支持变长的**Tuple**，那么**Page**的元数据需要保存每个**Slot**的长度（或者 **Slot** 的 **Offset**).
元数据从前往后存，**Tuple**从后往前存。当我们要插入一个**Tuple**的时候，一个个**Page**看过去，看看谁还有足够的空间（Tuple + Slot 能装得下）。插入成功后，返回**Page & Slot**号（**RID**），**RID**可以唯一标识一个**Tuple**.
![[./assets/Untitled 3.png|Untitled 3.png]]
![[./assets/Untitled 4.png|Untitled 4.png]]
读取的时候，如果**Page**不在内存，用我们的 buffer pool + 合适的 evict 策略，可以把**Page**弄到内存。然后根据 `Slot array + Slot number`，可以找到 Tuple 对应的 offset 和长度，从而读取。
**Tuple**往往分为两部分（如上右图），**Meta** 和 **Payload**, **Meta**包括 `timestamp` 和 `is_deleted`, 而**Payload**才是具体的数据。(注意 `tuple_meta.ts` 在 TableHeap 上）
注意，格式 (**Schema**) 是不存在 Tuple 里面的，因此各种解析**Tuple**的函数都要传**Schema**进去。
在 Bustub 中，Tuple 的实现方式是 `std::vector<char>`, 就相当于是可变长字节块，因此在赋值移动的时候，要尽可能使用 `std::move` 以及引用。

> 以上是 2021 年前 Bustub 要求实现的部分，仅作为背景知识。
- **Txn manager**是一个中心化的数据结构，包括所有 Transaction (`txn`)，以及每个**Tuple**对应的第一个 `undolog` 的指针。（相当于这一行 Tuple 对应的 Log 链表的头）
    - logs 存在 `txn` 内的一个 `vector` 中，logs 之间的关系有 log 内部的指针决定 (参考链表)

> 这里的指针是一个泛化说法，like iterator, 也就是 `<txn_id, log_id>`, 当你获得这样一个指针，通过 `txn_mgr` 得到事务，就可以访问到这个 `txn` 的 `vector_logs`, 就可以访问到这个 log.
`txn_mgr` 会保存一个哈希表来保存每一行 undolog 链表的头节点，哈希表储存 `<table_id, RID> → <txn_id, log_id>` 的映射。
这样设计的好处是，log 的构造是随着事务产生的，GC 是以事务为粒度的。存在 `txn` 中用连续空间储存 logs, 可以避免大量的内存操作。
注意，同一行的 undologs 内存上储存不是连续的，而是依靠指针连续储存。
- **Timestamp**, Bustub 将其分为两类：
    - Case 1: 如果第 63 位是 1, 那么是未提交的 `txn timestamp`
        - `txn` 的 `timestamp` 在未提交的时候是 `txn_id` (`txn_id` 的第 63 位永远是 1)
        - 在提交后，`txn` 的 `timestamp` 是 `commit_ts`
    - Case 2: 如果第 63 位是 0，那么是已经提交的 `txn` `commit_ts`
很显然，`undo_log.ts` 只能是已经提交的 `txn`. 而 `tuple_meta.ts` 则可以是 Case 1 & Case 2.
- **Undolog**包括 `modified bool vector`, `tuple`, `ts` , `is_deleted`.
    - 比如某个 `undolog` 的产生原因是，修改了 Tuple 的第 4，7 列，那么：
        - `undolog.tuple` 的长度为 2，值是修改之前的列
        - `modified bool` 的第 4, 7 个元素是 True, 其他元素是 False.
        - `ts` 在提交后为 `commit_ts`
        - `is_deleted` 为 False

> 建议实现一个从 undolog + 原 Schema 得到 `undolog.tuple.schema` 的工具函数。
![[assets/Untitled 7.png]]
最后一个 confused 的点在，`undolog.ts` 是什么意思？拿上面那张图的 A4 举例子，`ts = 4` 代表 TableHeap 上的 Tuple 已经 commit 了，`commit_ts` 是 4.
而 log 上的 `ts = 3` 则代表原来 Heap 上有一个 `commit_ts` 为 3 的 Tuple 被覆盖了，如果 `txn.read_ts` 为 3，那么就可以停止收集 undolog，把这个 undolog apply 之后就可返回了。
### Txn Manager & Watermark
MVCC 要维护全局的 `txn_mgr`，核心是一个水位 (Watermark). 因为 MVCC 需要维护大量的 logs，必须要 GC (垃圾回收). Watermark 记录了当前最晚的 `read_ts`, 比如是 20. 那么如果有一个 Log 是 40→30→25→15→10→5, 那么我们可以合理的 GC 10/5 这俩。用 `std::map` 实现即可。
### Seq Scan
首先实现一个根据现在的 Tuple 和 undolog 构建出原来的 Tuple 的工具函数 `ReconstructTuple`.
这个函数要重新构建一次 `std::vector<Value>` , 如果 `modified[i] = true`, 那么从 Tuple 中把对应的 `value[i]` 取出来，如果没有，则从原来的 Tuple 中把对应的 `value[i]` 取出来。这里注意用移动语义。
这里有个优化就是 `logs` 从后往前扫描，因为如果遇到了 `is_deleted = true` 的 log，前面的 logs 就都不用扫描了。
这里返回的是 `std::optional`, 构建失败的可能有：
- 遇到了 `is_deleted = true` 的 log A，而且 log A 之后并没有重新插入的 log, 也即这个 Tuple 最后被删除了
- Heap 上的被删了而 `undolog.size == 0`
然后实现 Seq scan, 分为三种情况：
- Heap 上面的 Tuple 是可以读取到的 (`tuple_meta.ts ≤ read_ts`) / Table Heap 上的 `tuple_meta.ts` 是当前的 `txn_id` (Txn Working…)
    - 返回 Table Heap 上的值，注意判断 `is_deleted`
- Table heap 上面不可读取 / 在被其他的 txn 读取 (根据第 63 位是否为 1 判断)，需要组装 `undologs`
    - 收集直到当前的 `undolog.ts < read_ts` 为止
**注意测试中有若干被注释的隐藏用例，需要自己思考答案然后加上去。**

> 这里要区分 `undo_link.has_value` 和 `undo_link.is_valid`, 只有二者都成立才真的存在这个 undolog.

> 这里要注意实现 `execution_common.cc` 中的那个 debug 函数，这也给我们平常开发一些参考，额外花大量的时间写一个辅助 debug 的函数是值得的！  
>   
> 后面多亏了每次打印的「现在的 Tuple + Undolog」，才能一路 debug 通关。  
### Insert
Insert 在一开始是很简单的，需要实现两部分：
- Insert 完为 `txn` 的 `WriteSet` 加上对应的 RID, 并且更新一下 `txn_mgr` 中的指针，让其指向空的 `undo_link`
    - Insert 的时候，注意将 `tuple.meta` 设置为 `txn_id`
- `txn_mgr` 在 `commit` 的时候要遍历 `WriteSet`, 把 `meta.ts` 从 `running txn_id` 的第 63 为为 1 的状态设置为真正的 `commit_ts`
### Update & Delete
这里真的要开始组装 `undolog` 了，二者都需要检查写-写冲突 (W-W Confilct)：
- 如果现在的 `tuple_meta.ts` 是 `running txn_id`，且不是当前 `txn_id`，那么 `Abort`
- 如果现在的 `tuple_meta.ts` 是 `commit_ts`，而且晚于现在的 `read_ts`，说明读取之后被人写过了，那么现在的结果是不准确的，所以 `Abort`
两者都需要成为 `pipeline-breaker`, 也就是在 `init()` 的时候把 `child.Next()` 调用到返回 `False` 的火山节点。
**Delete 要分 3 种情况讨论：**
- `tuple_meta.ts` 是当前的 `txn_id` (不可能 `delete` 后再 `delete` , 只能是先被 `update` / `insert`)
    - 已经被 `update` 创建了一个 undolog，因此需要**更新这个 undolog.**
        - 用现有的 `tuple` + 第一个 `undolog` + 上面写的 `reconstruct` 函数还原，然后更新这个 undolog 为 `modified vector = all true`，`tuple = tuple after reconstructing`
    - 这个 `tuple` 就是这个 `txn` 插入的，那么标记删除即可
    - 区分这两种情况的方法主要看有没有第一个 undlog, `insert` 的情况是一个也没有的～
- `tuple_meta.ts` 不是当前的 `txn_id`，那么插入一个 undolog 为 `modified vector = all true`，`tuple = origin_tuple` 的 undolog 即可
**Update 要分 3 种情况讨论**，注意这里使用 UpdateInPlace 而非删除后重新插入：
- `tuple_meta.ts` 是当前的 `txn_id` (不可能 delete 之后 update, 只能是 `update/insert`)
    - 之前可能被 `update` 创建了一个 undolog，我们要比对这次 `update` 新更新了什么
        - 如果是把以前更新过的列再更新一遍，那么不能写到 undolog
        - 如果以前这个列没有更新过，那么要加入到 undolog
        - 这需要对比现有的第一个 undolog 的 `modified_fields`
    - 这个 Tuple 就是当前 `txn` 插入的，那么更改 `heap.tuple` 即可
- `tuple_meta.ts` 不是是当前的 `txn_id`，那么对比 `update` 更新了什么然后生成 undolog 即可
    - 注意使用 `Value::CompareExactlyEqual` 进行判断
注意以上都要加入 WriteSet，都要更新 Index。

> 组装 undolog，判断哪些 column 修改以及插入 undolog 都可以提取出来作为工具函数。  
>   
> 此外，从这里开始，之前 Lab 3 的测试无法再通过了（限制 Tuple 是定长的），注意 DISABLE 测试以通过 CI。  
### Stop-the-world GC
实现一个 `clear_this_log_and_after` 函数即可。也就是根据 `watermark`，保留每个 undolog chain 的第一个小于 `watermark` 的 undolog，剩下的全删了。
这里删不是真的删，我们知道 undolog 物理连续存储是在 `txn` 的 `vector` 里的，也就是如果这个 `txn` 里面的所有 undolog 都可删的前提下，把这个 `txn` 删掉。由于 `txn` 的智能指针是 RAII 管理的，只要从 map 中移除，内存资源会自动释放。
这里有个 STL 糖，**我采取的方法是把被删掉的****`undolog.ts`****设置为****`INVAID_TS`**, 那么判断这个 `txn` 能不能删可以用 `std::all_of`, 写的很干净～
```C++
// if return true, delete from txn_map
auto TransactionManager:: CheckTxnInvalid (std::shared_ptr<Transaction>& txn) -> bool {
  If (txn->GetTransactionState () == TransactionState:: RUNNING ||
      Txn->GetTransactionState () == TransactionState::TAINTED) {
    Return false;
  }
	// 原来要用 for-if 语句
  Return std:: all_of (txn->undo_logs_. Begin (), txn->undo_logs_. End (),
  [](const UndoLog &undo_log) { return undo_log. Ts_ == INVALID_TS; });
}
```
### Consider primary key
考虑 Primary key (PKey) index 之后，最显著的变化是，每个 PKey 只能被插入哈希表一次，对应一个 RID.
所以如果 PK1 对应的 RID 被删除，再插入 PK1，那么不能分配新的 RID，否则 primary key index 就错了。我们知道关系型数据库的主键是不能随意修改的。
此外，还要求我们更改写-写冲突的判定条件。原来，Insert 是不和其他冲突的，现在考虑主键之后，就有可能冲突了。因为可能别人正在 `delete/update` 这个主键的 RID.
这里需要实现全新的并发控制，锁放在了 `txn_mgr` 为每个 `tuple` 保留的 `version_link` 中的 `in_progress` 字段。我们只要能原子的更新这个字段，相当于实现了行级别的锁。
注意，MVCC 的无锁只是读操作～写操作肯定还是要上细粒度的行级别的锁的。
![[assets/Untitled 8.png]]
仔细看上面的图和下面的箭头含义，这很重要！我们通过原子的更新 `version_link` 作为 `commit_point`.
这是 `LockRID` 实现行锁的代码，我们通过 update `version_link` + `check` 闭包的方式保证原子更新了 `in_progress` 字段。
这里足以看出闭包的强大。`in_progress == false` 代表没有人正在写，而 `origin_version_link->prev_ == version_link->prev_` 保证了没有 TOCTTOU 的问题。
```C++
Auto LockRID (RID rid, TransactionManager *txn_mgr) -> bool {
  Auto version_link = txn_mgr->GetVersionLink (rid);
  If (version_link. Has_value ()) {
    Return (txn_mgr->UpdateVersionLink (rid, VersionUndoLink{UndoLink{version_link->prev_}, true},
                                       [version_link](std::optional<VersionUndoLink> origin_version_link) -> bool {
                                         Return origin_version_link. Has_value () && !Origin_version_link->in_progress_ &&
                                                Origin_version_link->prev_ == version_link->prev_;
                                       }));
  }
  Return txn_mgr->UpdateVersionLink (
      Rid, VersionUndoLink{UndoLink{}, true},
      [](std::optional<VersionUndoLink> version_link) -> bool { return !Version_link. Has_value (); });
}
```
**Update & Delete 的修改：**
- 在最开始 LockRID，如果失败，而且拿着锁的不是本 txn，`Abort`
- 在 Index 更新环节避免更新 `pkey index`
**Insert 的修改：**
- 如果没有 primary index，之前 Insert 一次创建一个 RID 的操作可以继续，以下都不需要改
- 如果 RID 本来存在于 `primary index`，后来被删除，并且在本次 Insert 再次插入，那么 `Lock` 行，如果失败，且拿着锁的不是本 txn，`Abort`
    - 如果拿着锁的是本 txn，说明是本 txn 删除的 Tuple，`update_in_place` 即可
    - 如果 Lock 成功，说明不是本 `txn` 删除的 Tuple，这需要组装 `is_deleted = true` 的 undolog 插入并且更新 `VersionLink`
- 如果 RID 根本不存在，那么插入，Lock 行
此外，之前所有的和 undolog 有关的 `Get, Insert` 都要用 TA 给的有锁的函数。比如正确的获取 Undolog 的方法是：
```C++
While (undo_link. Has_value () && undo_link->IsValid ()) {
    Auto undo_log_opt = txn_mgr->GetUndoLogOptional (undo_link.Value ());
    If (undo_log_opt. Has_value ()) {
      // ...
      Undo_link = undo_log_opt->prev_version_;
    }
}
```
而不是：
```C++
While (undo_link. Has_value () && undo_link->IsValid ()) {
  Auto prev_txn = undo_link->prev_txn_;
  Auto prev_log = undo_link->prev_log_idx_;
  BUSTUB_ASSERT (txn_mgr->txn_map_. Count (prev_txn), "if txn undo value is valid, prev txn should be value");
  Auto undo_log_ptr = txn_mgr->txn_map_[prev_txn]->GetUndoLog (prev_log);
	// ...
  Undo_link = undo_log_ptr. Prev_version_;
}
```
以上在使用 `txn_map` 的时候没有 `lock`，会出问题。
最后的最后，要为 `txn_mgr` 的 `commit` 逻辑加上把 `in_progress` 都设置为 `false` 的操作。

> 思考：传统的 Write-Write Conflict 是否还有必要？  
>   
> 其实后半部分仍有必要：如果 Tuple 有  
> `commit_ts` 大于 `read_ts` 的，仍然是要 `Abort` 的
对于 IndexScan 和普通 Scan，印象中没有什么不同，可以抽象出一个函数减少代码。
这里我这个地方有点想不清楚。在更新完 `version_link` 之后，出现了 Table Heap 和第一个 `undolog` 都是 `ts = 3` 的话应该怎么办？中间有个很短的时间点似乎和原来的 Scan 逻辑对不上?
仔细观察，这时候 Heap 上的内容和 Log[0]的内容是一致的，如果一个 txn 看不到 Heap 上的内容，那么一定也会越过 Log[0]去找接下来的 Log, 而 Apply Log[0]的时候，由于和 Heap 完全一致，所以不会有任何变化。
### Primary key update
最后一关。首先把 Delete 和 Insert 的主逻辑抽到 `execution_common` 中，然后在 Update 的每次 `modified_fields` 组装好之后，就判断是否和 Pkey Interfere, 如果是的，那么先 Delete 再 Insert 即可。
最后 `concurrent_test` 的时候发现了一个 Bug，如果先 `LockRID` 再检查是否有读后修改，那么如果有读后修改导致的 W-W Conflict，没有实现 `Abort` 之前，要先 Unlock `in_progress` 字段再 `Abort`.