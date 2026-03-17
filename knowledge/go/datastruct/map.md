# map原理及面试问题

## map的底层原理

map是一种哈希表，可以通过键直接获得其对应的值。 哈希表要使用hash函数，通常情况下哈希函数的键的数量会远远大于值得数量，因此哈希函数是会发生冲突的。常见的哈希冲突的解决办法有三种

1. 开放寻址法:当产生哈希冲突时，回乡下一个索引值不为空的位置写入数据。 装载因子是当前已装载数量与总数量的比值，当装载因子超过70%时，哈希表的性能会急速下降。
2. 链地址法：使用数组加链表的结构（也可能会加入红黑树）。当出现哈希冲突时，将新的元素链接到前一个元素的末尾。

## go的map如何实现

> **https://cs.opensource.google/go/go/+/refs/tags/go1.26.1:src/internal/runtime/maps/map.go**
> **https://go.dev/blog/swisstable**

go的map基于Abseil的Swiss Table。

概念：

- slot：一个 key/value 对的存储位置
- group：8个slot组成的group, 包含一些控制字
- control word：一个8字节的字，用于指示slot是否为：空/删除/正在使用。如果正在使用，它会包含hash结果的低7位
- H1：hash结果的高57位
- H2：hash结果的低7位
- Table：一个完整的Swiss Table的哈希表，包含了一个或多个group。附加了若干元数据，用于控制增长
- Map：Map类型的顶级实现，包含了0或多个和table。hash结果的高位用于指示key属于哪个table
- Directory：map使用的一族table的数组

从本质上讲，该表的设计类似于传统的开放寻址哈希表。存储结构是一个Group的数组，中间穿插了一些控制字。

查找时使用哈希值来确定最初要检查的组。如果因为冲突，这个组里没有匹配项，就会按照探查（Probe）序列寻找下一个要检查的组。

Swiss Table允许利用额外的控制字（control word）来并行检查8个slot。

控制子中的每一个字节对应一个slot。在每个字节中：

- 用一个bit存储状态：空/正在使用/已删除
- 剩余7个bit存放该曹魏 key 的哈希值的低 7 位（H2）

在查找过程中，并行输入全部8个7-bit哈希片段进行比较，

探查（Probe）：

探查使用高57位（H1）作为索引，定位到groups数组中的某个组，采用二次探查，依次检查各个组。知道找到匹配或者有空槽的组。

删除：

探查会遇到空槽时停止。如果要删除的slot所在的group是完全满的，就**不能**将该slot标记为空，使用**墓碑（tombstone）**标记为已删除。如果本来就有空位就直接标记为空。插入时优先使用墓碑元素。墓碑在整体扩容时才会被清除。

扩容：

当组的数量发生变化时，才会进行扩容。map将内容分布到多个table中，每个table都是一个完整的hash表，但只负责一部分hash空间。扩容只针对单个table,因此可以实现增量扩容。

map初始只有一个table。在达到 **maxTableCapacity** 之前，扩容知识简单的把这个表换成容量翻倍的新表。超过这个阈值后，扩容会将这个表拆分成两个子表。

选择哪个表来操作使用的是可扩展哈希（extendible hasing）机制：

- 用哈希值的高几位作为索引，指向一个目录数组
- 目录中存放各个子表的指针
- 使用的位数 Map.globalDepth 会随表的增加而增加
- 例如：
  - 只有一个表时，用0位
  - 有两个表时用1位
  - 目录大小始终是 2^globalDepth
- 每个子表要记录自己创建时的深度，localDepth,当要分裂一个表且globalDepth==localDepth时，必须扩大目录

```go
// locate: src/internal/runtime/maps/map.go

type Map struct {
	// The number of filled slots (i.e. the number of elements in all
	// tables). Excludes deleted slots.
	// Must be first (known by the compiler, for len() builtin).
	used uint64

	// seed is the hash seed, computed as a unique random number per map.
	seed uintptr

	// The directory of tables.
	//
	// Normally dirPtr points to an array of table pointers
	//
	// dirPtr *[dirLen]*table
	//
	// The length (dirLen) of this array is `1 << globalDepth`. Multiple
	// entries may point to the same table. See top-level comment for more
	// details.
	//
	// Small map optimization: if the map always contained
	// abi.MapGroupSlots or fewer entries, it fits entirely in a
	// single group. In that case dirPtr points directly to a single group.
	//
	// dirPtr *group
	//
	// In this case, dirLen is 0. used counts the number of used slots in
	// the group. Note that small maps never have deleted slots (as there
	// is no probe sequence to maintain).
	dirPtr unsafe.Pointer
	dirLen int

	// The number of bits to use in table directory lookups.
	globalDepth uint8

	// The number of bits to shift out of the hash for directory lookups.
	// On 64-bit systems, this is 64 - globalDepth.
	globalShift uint8

	// writing is a flag that is toggled (XOR 1) while the map is being
	// written. Normally it is set to 1 when writing, but if there are
	// multiple concurrent writers, then toggling increases the probability
	// that both sides will detect the race.
	writing uint8

	// tombstonePossible is false if we know that no table in this map
	// contains a tombstone.
	tombstonePossible bool

	// clearSeq is a sequence counter of calls to Clear. It is used to
	// detect map clears during iteration.
	clearSeq uint64
}

```

## Q: sync.Map和普通的map有什么区别？

A：

sync.Map