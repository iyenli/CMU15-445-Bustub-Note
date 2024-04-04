# CMU15-445-Bustub-Note
Since Andy(Instructor of 15445) asks us not make project implementations public in exchange for making his course available to the public, the repository just include notes I wrote.

------------------------------------------------

由于教师Andy Pavlo要求不能将实现开源，因此本Repo只包含写的笔记。



## 环境

要注意 `master` 是 WIP 的，要选择 Release 中的压缩包版本。
用 VSCode debug, 运行→添加配置在 `.vscode/launch.json` 下设置 `"program": "${workspaceFolder}/build/test/xxx"` 即可。



## Lab 0

写一个 Snapshot based Trie Tree, 意义是可以并行的读取，任何一个写操作，都必须创建一个新的 Root，然后原子的把当前的 Trie DB 的树根换成新的。
- 如果是读取，那么原子的获取当前的树根，接下来的所有节点（因为没有 Inplace 的修改），都不会被并行性所影响。
- 如果是写入，那么原子的获取当前树根，（要写哪个节点，我们假设根到节点的 Path 为 P）然后复制一个新的树根（指向原来树根的所有孩子，除了 P），然后一路把 P 给覆盖掉，然后再原子的替换当前树根

后面还有个 C++转换大小写的小练习。用 `std::transform` 可以优雅的完成：

```C++
if (expr_type_ == StringExpressionType::Lower) {
  std::transform(val.begin(), val.end(), std::back_inserter(res), [](unsigned char c) { return std::tolower(c); });
} else if (expr_type_ == StringExpressionType::Upper) {
  std::transform(val.begin(), val.end(), std::back_inserter(res), [](unsigned char c) { return std::toupper(c); });
} else {
  throw Exception("Not support string expression type");
}
```
