
会者不难👌（逃
我们这里主要做的是 Optimizer 和 Executer 两部分。Parser 正如编译器做的那样，将文本转换为内部数据结构 (抽象语法树 AST)，Binder 将抽象语法树中的节点和数据库中的实际对象关联起来，比如 `select * from table1;`，binder 就会找到 table1，用它的所有 Column 代替这个 `*`。Planner 接受 AST，生成一个执行计划（火山模型的树），然后 Optimizer 则将逻辑计划通过一些规则优化为物理计划。
![[./assets/Untitled 2.png|Untitled 2.png]]
### 准备工作
需要了解火山模型和 `Init-Next` 的工作原理。善于利用本地和 Web 网站上的 EXPLAIN 功能，知道给定一个查询语句，是怎么转换成火山模型的，又是如何优化的。
```C++
Bustub> EXPLAIN SELECT * FROM __mock_table_1;
=== PLANNER ===
Projection { exprs=[\ #0 .0, #0 .1] } | (__mock_table_1. ColA: INTEGER, __mock_table_1. ColB:INTEGER)
MockScan { table=__mock_table_1 } | (__mock_table_1. ColA: INTEGER, __mock_table_1. ColB:INTEGER)
=== OPTIMIZER ===
MockScan { table=__mock_table_1 } | (__mock_table_1. ColA: INTEGER, __mock_table_1. ColB:INTEGER)
```
还需要仔细读一下给定的 Sample Executors，举个例子讲解一下：
```C++
ProjectionExecutor:: ProjectionExecutor (ExecutorContext *exec_ctx, const ProjectionPlanNode *plan,
    std::unique_ptr<AbstractExecutor> &&child_executor)
    : AbstractExecutor (exec_ctx), plan_(plan), 
    Child_executor_(std:: move (child_executor)) {}
Void ProjectionExecutor:: Init () {
  Child_executor_->Init ();
}
Auto ProjectionExecutor:: Next (Tuple *tuple, RID *rid) -> bool {
  Tuple child_tuple{};
  // Get the next tuple
  Const auto status = child_executor_->Next (&child_tuple, rid);
  If (! Status) {
    Return false;
  }
  // Compute expressions
  std::vector<Value> values{};
  Values.Reserve (GetOutputSchema (). GetColumnCount ());
  For (const auto &expr : plan_->GetExpressions ()) {
    Values. Push_back (expr->Evaluate (&child_tuple, child_executor_->GetOutputSchema ()));
  }
  *tuple = Tuple{values, &GetOutputSchema ()};
  Return true;
}
```
`exec_ctx` 是每次执行都会带的上下文，包括 `transaction_manager` (in project \ #4 ), `table_heap` . `child_executor` 是火山模型子节点。`plan` 则是当前 `executor` 执行的上下文，如算子。
`Init` 函数只会调用一次，`Bustub` 会调用根节点的 `Init`, 由父节点调用子节点的 `Init`. 然后当父亲节点调用 Next, ProjectionExecutor 调用子节点的 `Next` 函数，调 `AbstractExpression::Evaluate` 函数。`AbstractExpression` 是一个抽象类。
- 比如为 `ColumnExpression`，我们传入 Tuple 到 `Evaluate`，就会返回这个 Tuple 的对应的列值。
- Expression 为 `CompareExpression`, 那么这个 Expression 一定是两个子 Expression 初始化的
    - `Evaluate` 的时候将 Tuple 传递到两个子 Expression 中 `Evaluate`，再用父 Expression 中的 `Logical` 算子比较。
    - Seq Scan 中的 `predicate` 就是由一个 `CompareExpression` 子节点为 `ColumnExpression` + `ConstantExpression` (无论传入什么都返回常量) 实现 `where col1 = 1` 的
Next 返回 true 代表还有下一个值，父亲还要调用 Next, 且 Tuple 和对应的 RID 存在传入的指针中。
在此之前，还需要了解存储的基本结构：（这一段和 Project \ #4中的overview是重复的 ）
**Heap**是**Table**的磁盘储存格式，包括若干个**Page**指针, **Tuple**是关系型数据库的**一行**，而**RID**是找到这一行的索引，包括**Page ID 和 Slot ID**.
一个**Page**里面包含多个**Slot**, 如果要支持变长的**Tuple**，那么**Page**的元数据需要保存每个**Slot**的长度（或者 **Slot** 的 **Offset**).
元数据从前往后存，**Tuple**从后往前存。当我们要插入一个**Tuple**的时候，一个个**Page**看过去，看看谁还有足够的空间（Tuple + Slot 能装得下）。插入成功后，返回**Page & Slot**号（**RID**），**RID**可以唯一标识一个**Tuple**.
![[./assets/Untitled 3.png|Untitled 3.png]]
![[./assets/Untitled 4.png|Untitled 4.png]]
读取的时候，如果**Page**不在内存，用我们的 buffer pool + 合适的 evict 策略，可以把**Page**弄到内存。然后根据 `Slot array + Slot number`，可以找到 Tuple 对应的 offset 和长度，从而读取。
我们可以看出，只有查询表中的 Tuple，RID 才有意义。这也是为什么上面的 ProjectionExecutor 没有设置 RID 的原因。
Tuple 的格式是 Schema, 因为 Tuple 是由字节数组组成的，本身不含格式。因此需要传入一个 Schema 语义才完整。
最后，`exec_ctx` 中的全局变量 `catalog` 很关键，`catalog` 维护了若干哈希表，保存了 `table_id` 和 `table_name` 到 `table_info` 的映射关系。`table_id` 新建 `table` 时自动分配，`table_name` 则由用户指定。此外，`table_id` 和 `table_name` 到 `index_info` 之间也可以相互转换。
想要插入，得到 Tuple, 需要根据 `plan` 中的 `table_id / table_name` 得到对应的 `table_info`, 得到对应的 `heap`.
总结一下上面的 Projection Executor, 关键还是对每一列使用预先组装好的 planner 中的 expression 来进行修改，然后组装回 Tuple 进行输出：
```C++
std::vector<Value> values{};
Values.Reserve (GetOutputSchema (). GetColumnCount ());
For (const auto &expr : plan_->GetExpressions ()) {
  Values. Push_back (expr->Evaluate (&child_tuple, child_executor_->GetOutputSchema ()));
}
```
### Access Method Executors
Seq scan: 只会出现在叶子结点，不需要调用子节点的 `Next` , 要了解基本的迭代器用法, TableItrator 实现了 `auto operator++() -> TableIterator &;` , 这是 `prev++`.
如果 `tuple_meta.is_deleted = true`, 那么就不返回。
`pridicate` 的用法：
```C++
If (plan_->filter_predicate_) {
  If (auto value = plan_->filter_predicate_->Evaluate (tuple, GetOutputSchema ());
      value.IsNull () || !Value. GetAs<bool>()) {
    Continue;
  }
}
```
Update: Tuple 的组装请看构造函数，是由 `std::vector<Value>` + `schema` 得到一个 `vector<char>` . 在 Project 3，需要先删除再插入。Update 的值是由 `plan.targe_expressions` Evaluate 得到的。而 tuple 则是由子节点 `while (child.Next())` 得到的。
```C++
For (const auto &target_expression : plan_->target_expressions_) {
  Values. Emplace_back (target_expression->Evaluate (tuple, 
													  Child_executor_->GetOutputSchema ()));
}
```
三个写操作的返回值都是一个单值 Tuple, 表示成功了多少行。所以这里的 Schema 不能用 `this→GetOutputSchema()` . 我们维护一个 `success_number` , 最后组装返回即可。
```C++
std::vector<Value> values{Value{INTEGER, ret}};
*tuple = Tuple{values, &GetOutputSchema ()};
```
最后，三个写操作都只返回一次，因此可以用一个值记录做过没有，第二次调用 Next 的时候，如果发现做过，返回 False. 三个写操作都要注意更新索引，不然后面可能会有坑。
Insert: 和 Update 一样。注意 `InsertTuple` 返回的是 `std::optional` , `if has_value` 才成功。
Delete: 和 Update 一样。注意删除的方法是将 `is_deleted` 设置为 true.
**IndexScan:** Index 的结构是 Key→RID, 通过 class 成员 `std::vector<RID> res` 可以储存这个 Key 索引到的所有 RID, 再通过 Cursor 来遍历 RIDs, 每次 emit 一个 Tuple.
```C++
Auto htable = 
  dynamic_cast<HashTableIndexForTwoIntegerColumn *>(index_info_->index_. Get ());
htable->ScanKey (Tuple{std::vector<Value>{plan_->pred_key_->val_}, &schema}, 
  &res_, exec_ctx_->GetTransaction ());
```
然后需要把符合条件的 SeqScan 优化为 IndexScan, 这需要先将 filter “push down”到 scanner. 在原本的火山中是这样的：
```C++
Filter { predicate=(\ #0 .0=1) } | (t1. V1: INTEGER, t1. V2: INTEGER, t1. V3:INTEGER)
SeqScan { table=t1 } | (t1. V1: INTEGER, t1. V2: INTEGER, t1. V3:INTEGER)
   
// 下推之后
SeqScan { table=t1, filter=(\ #0 .0=1) } | (t1. V1: INTEGER, t1. V2: INTEGER, t1. V3:INTEGER)
// 如果 Column 0 有 Index, 我们的优化器应该优化一下：
IndexScan { index_oid=0, filter=(\ #0 .0=1) } | (t1. V1: INTEGER, t1. V2: INTEGER, t1. V3:INTEGER)
```
看 `Optimizer::OptimizeCustom(const AbstractPlanNodeRef &plan)` 函数已经把所有的规则都 Apply 了。这个函数只对根节点调用一次，因此需要 `OptimizeSeqScanAsIndexScan` 内对所有子节点调用相同的优化函数。
参考讲解一下 `OptimizeMergeFilterScan` :
```C++
// 递归调用，所有的 Optimize rule 都有，照抄就好
std::vector<AbstractPlanNodeRef> children;
For (const auto &child : plan->GetChildren ()) {
  Children. Emplace_back (OptimizeMergeFilterScan (child));
}
// 传入的是一个 const reference, 为了修改，需要 clone 一个一模一样的
// 注意，孩子节点需要是「优化后的」
Auto optimized_plan = plan->CloneWithChildren (std:: move (children));
// 这里优化的是下推 Filter，要求自然是当前节点为 Filter 下面的节点为 SeqScan
// 类推一下就知道，我们要写的这里应该是 SeqScan
If (optimized_plan->GetType () == PlanType::Filter) {
	// 既然知道是 Filter 了，那么 dynamic_cast 可以转换为正确的子类啦
  const auto &filter_plan = dynamic_cast<const FilterPlanNode &>(*optimized_plan);
  BUSTUB_ASSERT (optimized_plan->children_. Size () == 1, "must have exactly one children");
  
  // 对唯一的孩子，如果是 SeqScan 而且 predicate 没有被占用，那么 push down 一下
  Const auto &child_plan = *optimized_plan->children_[0];
  If (child_plan.GetType () == PlanType::SeqScan) {
    const auto &seq_scan_plan = dynamic_cast<const SeqScanPlanNode &>(child_plan);
    If (seq_scan_plan. Filter_predicate_ == nullptr) {
      return std::make_shared<SeqScanPlanNode>(filter_plan. Output_schema_, seq_scan_plan. Table_oid_,
                                               Seq_scan_plan. Table_name_, filter_plan.GetPredicate ());
    }
  }
}
```
类比一下，我们要写的逻辑是：
- 1) 如果是 Seq Scan, 强转为 SeqScanPlanNode
- 2) 如果 predicate 不为空，而且是 CompareExpression, 而且算子为 Equal, 那么强转 CompareExpression
- 3) 如果左边/右边子节点是 ColumnExpression 且符合 IndexMatch 要求，那么返回 IndexScan
```C++
Auto index_info = MatchIndex (seq_scan_plan. Table_name_, column->GetColIdx ());
If (index_info. Has_value ()) {
  return std::make_shared<IndexScanPlanNode>(seq_scan_plan. Output_schema_, seq_scan_plan. Table_oid_,
                                             std::get<0>(index_info.Value ()), nullptr, constant);
}
```
注意这里的 tuple get 方法的模板类很有意思～ `std::get<0>` 好处是编译的时候就能确定是取第 0 个元素，我们要的就是 `index_id`.
### Aggregation & Join Executors
**Aggregation**为输入的 Tuple 执行聚合计算，这里只要简单的纯内存哈希表支持就好 (不需要 Buffer Pool). 这是第一个 Pipeline breaker, 也就是说在 Init 的时候要把孩子节点 `Next` 完，**这里有个坑点在 Init 是可能多次被调用的，一定要保证 Init 函数的可重入性。**
理解 `count` & `count star` 的区别：`count` 可能会返回 Null, 当没有符合条件的行时，会返回 Null Integer. 而 count star 则会返回 0.
聚合是诸如这样的语句：
```C++
EXPLAIN SELECT COUNT (colA), MIN (colB) FROM __mock_table_1 GROUP BY colA;
```
下面往往是 Emit Tuple 的数据产生节点。我们得到的结果是这样的：
- ColA = Va1 → (colA 有值的行树，这些 Tuple 中 colB 的最小值）
- ColA = Va2 → (colA 有值的行树，这些 Tuple 中 colB 的最小值）
- …
因此帮我们维护了一个哈希表，Key 是 Group Bys Expression 算出来的 Columns, Value 在这句话里是 (Col A, Col B), 然后哈希表被包装成 `SimpleAggregationHashTable aht`，当我们执行 `aht.Insert(Key, Value)`, `aht` 找到 Key 对应的条目 (`count_colA, min_colB` ), 执行 `count + 1 and colB = min(colB, value B)` .
在 Init 中，我们 Next 子节点，然后通过 `Expressions` 造出 Key, Value, 插入 `aht` 中。在 Next 中，我们根据 Key 一行一行 Emit 出去即可。这里可以用 `std::back_inserter`
```C++
std::vector<Value> values;
Std:: move (aht_iterator_. Key (). Group_bys_. Begin (), 
	 Aht_iterator_. Key (). Group_bys_. End (), std:: back_insert_iterator (values));
Std:: move (aht_iterator_. Val (). Aggregates_. Begin (), 
	Aht_iterator_. Val (). Aggregates_. End (), std:: back_insert_iterator (values));
*tuple = Tuple{std:: move (values), &GetOutputSchema ()};
```
上面提到，保证 Init 的重入性，要在 Init 中先调用 `aht.clear()` 再插入函数，这很重要。
最后，要注意如果表是空的，我们不能返回空行。这要求至少 aht 要有一个条目。Init 后如果一个都没有，手动插一个初始值进去。可以重用 `AddEmpty` 函数。
NestedLoopJoin 实现的是 Join 语句。多说一句，Join 是很多数据库操作的瓶颈（道听途说来的），实现这个 NestedLoop 的时候总觉得在犯罪😟
这里补充下关键的 INNER JOIN 和 LeftOuterJoin 的区别。
`INNER JOIN` 返回两个（或多个）表中有匹配的记录。如果一行在连接的两边都有匹配，它才会出现在结果集中。假设我们有两个表，一个是员工表 `Employees` 和一个是部门表 `Departments`。
```Plain
+------------+-----------+------------+
| EmployeeID | Name      | Department |
+------------+-----------+------------+
| 1          | Alice     | HR         |
| 2          | Bob       | IT         |
| 3          | Charlie   | Sales      |
+------------+-----------+------------+
+------------+-----------+
| DeptName   | Manager   |
+------------+-----------+
| HR         | Eve       |
| IT         | Frank     |
| Marketing  | Grace     |
+------------+-----------+
```
我们的 SQL 语句可能看起来像这样：
```SQL
SELECT Employees. Name, Departments. DeptName, Departments. Manager FROM Employees
INNER JOIN Departments ON Employees. Department = Departments. DeptName;
```
结果集将会是：
```Plain
+--------+-----------+---------+
| Name   | DeptName  | Manager |
+--------+-----------+---------+
| Alice  | HR        | Eve     |
| Bob    | IT        | Frank   |
+--------+-----------+---------+
```
注意 "Sales" 部门和 "Marketing" 部门没有出现在结果集中，因为 "Sales" 部门没有匹配的经理，而 "Marketing" 没有匹配的员工。
`LEFT OUTER JOIN` 返回左表（即 `LEFT JOIN` 关键字左边的表）的所有记录，以及右表中匹配的记录。如果左表的行在右表中没有匹配，则结果集中右表的部分将包含 NULL。
使用上面同样的表，如果我们执行一个 `LEFT OUTER JOIN` 查询来找出所有员工及其部门经理，即使某些员工没有部门经理也要显示出来，我们的 SQL 语句可能看起来像这样：
```SQL
SELECT Employees. Name, Departments. DeptName, Departments. Manager FROM Employees
LEFT OUTER JOIN Departments ON Employees. Department = Departments. DeptName;
```
结果集将会是：
```Plain
+---------+-----------+---------+
| Name    | DeptName  | Manager |
+---------+-----------+---------+
| Alice   | HR        | Eve     |
| Bob     | IT        | Frank   |
| Charlie | Sales     | NULL    |
+---------+-----------+---------+
```
这次，Charlie 被包括在内，尽管他的部门没有对应的经理。Charlie 的部门是 "Sales"，但是在 `Departments` 表中没有 "Sales" 部门的经理，所以经理列显示为 NULL。
这里很容易写成 (Ref [https://zhuanlan.zhihu.com/p/587566135](https://zhuanlan.zhihu.com/p/587566135) 我们需要知道怎么样不行～)：
```C++
While (left_child->Next (&left_tuple)){
    While (right_child->Next (&right_tuple)){
        If (left_tuple matches right_tuple){
            *tuple = ...;   // assemble left & right together
            Return true;
        }
    }
}
```
Right child 在 left child 的第一次循环中就被消耗完了，之后只会返回 false. 那如果这样：
```C++
While (left_child->Next (&left_tuple)){
    For (auto right_tuple : right_tuples){
        If (left_tuple matches right_tuple){
            *tuple = ...;   // assemble left & right together
            Return true;
        }
    }
}
```
问题在于，一个 Left Tuple 万一匹配了很多很多 Right Tuple，这里无法反映了。所以解决方案是：
1. Right Tuple 必须存起来，还要有一个 cursor 指示遍历到哪里了
2. 当前的 Left Tuple 也必须存起来
这里，我们可能每次都会重新 Init 并且 Next RightTuple, 因此正如我们上面重复的那样，每个 Init 都应该是可以重入的。
为了实现 LEFT INNER, 需要记录当前 Left tuple, 当前 Left tuple 所需要匹配的下一个 Right tuple，当前 Left tuple 是否匹配过。在全部的 Right tuple 匹配完成之后，如果没有匹配过，而且是 Left Inner，那么需要发射一个 Left + Null right 出去。
```C++
std::vector<Value> empty_right_values;
For (const auto &col : right_executor_->GetOutputSchema (). GetColumns ()) {
  Empty_right_values. Emplace_back (ValueFactory:: GetNullValueByType (col.GetType ()));
}
for (uint32_t i = 0; i < left_executor_->GetOutputSchema (). GetColumnCount (); i++) {
  Values. Emplace_back (curr_left_. First.GetValue (&left_executor_->GetOutputSchema (), i));
}
Std:: move (empty_right_values.Begin (), empty_right_values.End (), std:: back_insert_iterator (values));
```
是否可以 Join 的判断：
```C++
If (auto value = plan_->predicate_->EvaluateJoin (&curr_left_. First, left_executor_->GetOutputSchema (),
&(curr_right_. First), right_executor_->GetOutputSchema ()); !Value.IsNull () && value. GetAs<bool>())
```
我真的很喜欢 `Golang` 的 `if` 判断…感谢 C++17！上面那篇文章还提到了 RisingLight 的 Rust 实现，确实很牛…此外，`Golang` 也可以优雅的实现，而不需要麻烦的 left_tuple, right_cursor, left_matched 这些莫名其妙的 flag…
```Rust
// Rust
Pub async fn Execute (){
    For left_tuple in left_table {
        For right_tuple in right_table {
            If matches {
                Yield AssembleOutput ();
            } // 当执行到 yield 时，函数会暂时中断，从生成器回到调用者
        } // 而调用者再次进入生成器时，可以直接回到上次中断的地方
    } // 无栈协程和异步编程
}
```
```Go
Func Executor (out_ch, left_ch, right_ch chan Tuple) {
    For left_tuple := range left_ch {
        For _, right_tuple := range right_tuples {
            If matches {
                Out_ch <- AssembleOutput ();
            } // coroutine + channel
        } // 也是优雅至极呀～
    }
}
```
### **HashJoin Executor and Optimization**
我们需要实现一个哈希表，JoinKey→tuple, 这里可以参考一下 `SimpleAggregationHashTable` , 为了让哈希表可以由 JoinKey 当 Key, 需要实现哈希函数和==函数。
```Go
Struct JoinKey {
  std::vector<Value> join_keys_;
  Auto operator==(const JoinKey &other) const -> bool {
    Return std:: equal (join_keys_. Begin (), join_keys_. End (),
	   Other. Join_keys_. Begin (), [](const auto& a, const auto& b) { 
		   return a.CompareEquals (b) == CmpBool:: CmpTrue; 
		 });
  }
};
Namespace std {
template <>
struct hash<bustub::JoinKey> {
  Auto operator ()(const bustub:: JoinKey &join_key) const -> std:: size_t {
    Size_t curr_hash = 0;
    For (const auto &key : join_key. Join_keys_) {
      If (! Key.IsNull ()) {
        Curr_hash = bustub::HashUtil:: CombineHashes (curr_hash,
						        Bustub::HashUtil:: HashValue (&key));
      }
    }
    Return curr_hash;
  }
};
}  // namespace std
```
对每个 `right_tuple`, 都需要 `right_exprs` 计算出 JoinKey 然后插入。对每个 `left_tuple` , 都需要计算出 JoinKey 之后，和哈希表对比，得到 RightTuple 集合，逐个组装 emit.
相信到这读者通过 `std::value` 组装 Tuple, 区分 `this→GetOutputSchema` 和 `child_executor→GetOutputSchema` 已经非常熟练了～
`OptimizeNLJAsHashJoin` 就是把 NestedLoopJoin 转换为 HashJoin 的过程了。主要的结构和其他 Optimizer 是一样的，我们需要进行如下优化：
```Go
 NestedLoopJoin { type=Inner, predicate=((\ #0 .0=\ #1 .0) and ( #0 .1= #1 .2)) } 
   SeqScan { table=test_1 }                                           
   SeqScan { table=test_2 }
   
 HashJoin { type=Inner, left_key=[\ #0 .0, #0 .1], right_key=[ #0 .0, #0 .2] } 
   SeqScan { table=test_1 }                                             
   SeqScan { table=test_2 } 
```
原来是 predicate (NLJ 中，我们直接用 `predicate.Evaluate` 决定要不要 Join), 我们需要转换为 Left Key & Right Key. 这里显然用递归过程是更合适的。
```C++
auto IsAndEqual (const AbstractExpression *expr, std::vector<AbstractExpressionRef> &left_key_expressions,
                std::vector<AbstractExpressionRef> &right_key_expressions) -> bool
```
- 如果本 Expression 是 And, 那么递归左子树和右子树，如果它们是 Equal / And 表达式，则返回 true
- 如果不是 And, 检查是不是 Equal 表达式，如果是，检查 Equal 表达式的左子树和右子树
    - `left_child->GetTupleIdx() == 0 && right_child->GetTupleIdx() == 1` 表示左子树来自左 Tuple, 右子树来自右 Tuple
    - 把正确的 Child (到这里肯定是 ColumnExpression) 放到对应的 `left/right_key_expressions`
### **Sort + Limit Executors + Top-N Optimization**
**Limit**: 额外维护一个 cursor, 到 Limit 返回 false
**Sort**: Pipeline breaker, 先弄到数组里，sort, 用 cursor 挨个 emit 出去。注意 order_by 和 strcmp 是一样的，只有相等才比下一个。使用 `column::CompareGreaterThan / column::CompareLessThan`
似乎要处理 IsNull 的情况？没有很关注 IsNull 有没有测试用例…
```C++
If (column1.CompareGreaterThan (column2) == CmpBool::CmpTrue) {
  Return order_by. First == OrderByType:: DESC;
}
If (column1.CompareLessThan (column2) == CmpBool::CmpTrue) {
  Return order_by. First != OrderByType:: DESC;
}
```
**TopK**: Sort + Limit-K 可以优化一下。大概就是原来是 `nlog(n)` 的时间复杂度，现在用 `nlog(K)` 就可以了。用的是堆来完成。这里的优化计划是最简单的，和下推几乎一样，就是 Limit 的儿子是 Sort，那么就优化为 TopN 即可。这里的 Compare 函数可以复用 Sort 中的。
如何声明一个自定义比较函数的 Priority Queue:
```C++
std::priority_queue<std::pair<Tuple, RID>, 
				std::vector<std::pair<Tuple, RID>>, decltype (compare)> pq (compare);
```
### 窗口函数
![[assets/Untitled 5.png]]
和 Agg 不同，窗口函数会保持行之间的关系（或者仅进行一次排序）不变，输出每一行以及其统计信息（如这一行的某一列的排序，这一行某一列的累加综合，计算移动平均等）。我们被要求支持：
- Partition by: 窗口，比如 Ranking 说的是在这个 partition 内的排名
- Order by: partition 内的顺序，先分块，再排序
- Window frames: 要考虑的窗口长度等，见上
```C++
SELECT user_name, dept_name, AVG (salary) OVER \
(PARTITION BY dept_name ORDER BY user_name ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) \
FROM t;
+-----------+-----------+-----------------------------+
| user_name | dept_name | salary                      | // origin salary
+-----------+-----------+-----------------------------+
| a         | dept1     | 150                         | // 100 -> 100 + 200 / 2 
| b         | dept1     | 200                         | // 200 -> 600 / 3
| c         | dept1     | 250                         | // 300 -> 500 / 2
| d         | dept2     | 75                          | // 50 -> 150 / 2
| e         | dept2     | 70                          | // 100 -> 210 / 3
| f         | dept2     | 80                          | // 60 -> 160 / 2
+-----------+-----------+-----------------------------+
```
以上，首先输出按照 `dept_name` 分块，块内按照 `name` 排序，然后 `window size` 为 1 前 1 后。ORDER BY 子句也保证一样，不会出现如下 SQL 语句：  
  
`SELECT SUM(v1) OVER (ORDER BY v1), SUM(v1) OVER (ORDER BY v2) FROM t1;`
正常来说，有 ORDER BY 子句的时候，应该是先分 partition, partition 内部再进行排序，而且排序方法会影响到 `window frame` 计算的准确性。但是本 Lab 不要求输出行的顺序（因此也无法要求 `window frame`, 这降低了很多难度），所以可以先进行全局排序，再执行 `window function` 在 partition 内进行聚合。
注意，如果有 ORDER BY 子句，那么默认的 `window_size` 为 partition 开始到现在，而没有 ORDER BY 子句，默认的 window_size 为整个 partition. 下面两个例子很好的说明了这一点：
```C++
SELECT user_name, dept_name, AVG (salary) // 有 Order By 子句的前提
OVER (PARTITION BY dept_name ORDER BY user_name) FROM t;
+-----------+-----------+-----------------------------+
| user_name | dept_name | salary                      |
+-----------+-----------+-----------------------------+
| a         | dept1     | 100                         | // 100 / 1
| b         | dept1     | 150                         |
| c         | dept1     | 200                         | // 600 / 3
| d         | dept2     | 50                          | // 50 / 1
| e         | dept2     | 75                          |
| f         | dept2     | 70                          | // 210 / 3
+-----------+-----------+-----------------------------+
```
```C++
SELECT user_name, dept_name,AVG (salary) OVER (PARTITION BY dept_name) FROM t;
+-----------+-----------+-----------------------------+
| user_name | dept_name | salary                      | // 没有 order by 子句
+-----------+-----------+-----------------------------+
| a         | dept1     | 200                         | // 600 / 3
| b         | dept1     | 200                         |
| c         | dept1     | 200                         | // 600 / 3
| e         | dept2     | 70                          | // 210 / 3
| d         | dept2     | 70                          |
| f         | dept2     | 70                          | // 210 / 3
+-----------+-----------+-----------------------------+
```
- 先利用 `sort_executor` 中的函数进行排序（注意检查所有 `window_function` 下的 `order_by` 是否一致，如果不一致要抛出异常）
- 需要 `window_functions.size()` 张哈希表记录聚合结果：
```C++
std::vector<std::unordered_map<AggregateKey, Value>> 
															Mp (plan_->window_functions_. Size ());
```
`vector` 的长度是 `window_function` 的数目。
`plan_->columns_` 控制最终输出的 Tuple，如果是 `ColumnExpression` , 那么通过 Evaluate 方法取出原 Tuple 对应的列，若是 Placeholder, 那么 `window_function` 一定存在，对每个 `window_function`, 都维护一张哈希表，Key 是 Partition 的值，而 Value 则是聚合结果（如排名，平均值等）。
对于 `window_functions[i]`, 获取 PartitionKey：
```C++
std::vector<Value> partition_values;
For (const auto &partition_expr : plan_->window_functions_. At (i). Partition_by_) {
  Partition_values. Emplace_back (partition_expr->Evaluate (&tuple_rid. First, child_executor_->GetOutputSchema ()));
}
```
获取用于聚合的 Key:
```C++
Auto attend_value = plan_->window_functions_. At (i).
			Function_->Evaluate (&tuple_rid. First, child_executor_->GetOutputSchema ());
			
// 根据 attend_value 完成聚合
Mp.At (window_function_idx)[AggregateKey{partition_values}] = ...
    
```
- 对于 Rank 而言，必然会有需要排名的列的 ORDER BY 子句，所以只需要按顺序分发第一第二第三名吗…？不是的，因为相同数字要求排名一致，如相同 Partition A 中有五个元素 1, 2, 2, 3, 3, 排名应当为 1, 2, 2, 4, 4 (纸面排名), 而非 1, 2, 3, 4, 5 (真实排名).
    - Workaround：需要一张额外的哈希表`rank_cache`记录从`AggregateKey → std::pair<Value, Value>` 的映射，`first`为真实的排名（每次递增），而`second`为当前纸面排名对应的 Value.
    - 举个例子，对于第二个 2 来说，`mp[A] = 2`, `rank_cache[A] = {3, 2}`. 当走到第一个 3 的时候，发现和`rank_cache[A].second`不一致，于是将`rank_cache[A].first + 1`作为`rank_cache[A].first` & `mp[A]`.
- 最后的问题就是处理 Sorted 和 Non-Sorted 啦，如果是 Sort 过的，那么每处理完一行，就要把`mp[AggKey]`的值写到对应的列上，如果没有 Sort，就要等所有的行处理完，再写`mp[AggKey]`