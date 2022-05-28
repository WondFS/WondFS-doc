## 功能描述

文件系统中内置的KV存储是用于加速 inode 的查找，以目录的inode+文件名的形式作为 Key 值，查询KV存储中的 Value 值，即获得该目录下该文件的 inode，从而实现快速解析路径的功能。

### 技术选型

由于 flash 闪存的物理特性，写入操作只能在空的页单元中进行，页单元在写入之前需要擦除，擦除是以块为单位进行的。所以 flash 不支持本地更新的数据结构，相反 flash 对于异地更新的数据结构支持较好。同时，因为 flash 以较大的页单元写入，通常为 4KB，所以适合将小数据积累成一定规模的数据后再写入。

基于上面的理由，我们考虑采用 Log Structured Merge Tree (LSM-Tree) 作为我们的 KV 存储系统的索引，相比于传统的 B-Tree，LSM-Tree 具备有以下特性：

1. LSM-Tree 具备批量写入特性，能够将小数据的 random write 转化为 sequence batch write 来适配 flash 的特性，从而提高写性能。
2. 以 append-only 的形式添加数据，而不是 update-in-place，这样的形式可以避免因为一小部分数据的修改引起 flash 中大量的 page 数据进行迁移。
   
### Log Structured Merge Tree 实现

#### 基本思路

LSM-Tree 的基本思路是在内存中累计一定量的写入数据（这部分数据被称为 Memtable）后，再将这部分数据批量写入 flash 中 （写入的数据被打包为单独的 SSTable 文件）。为了提高读性能，LSM-Tree 会确保 SSTable 文件都是有序的，以支持快速的二分查找或者指数查找。在将 Memtable 的数据以 SSTable 文件的形式写入时，LSM-Tree 会先将这部分数据进行排序，以保证从内存写入的 SSTable 文件是有序的。同时，当累计一定量的 SSTable 时，会将一部分的 SSTable 文件进行归并排序，以合并删除操作，减少每次需要查询的文件数量。由于内存数据的写入和归并排序，flash 上的结构最终会变成有序的层级结构，并且不同的 SSTable 之间的 Key 范围不会发生重叠。

具体形式可参考下图，但由于文件系统本身的特性，在 WondFs 中没有实现 immutable memtable 以及 block cache。
![](https://img-blog.csdnimg.cn/20210419113220640.png?x-oss-process%3Dimage%2Fwatermark%2Ctype_ZmFuZ3poZW5naGVpdGk%2Cshadow_10%2Ctext_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpYW9kaWRhZGFkYQ%3D%3D%2Csize_16%2Ccolor_FFFFFF%2Ct_70)

#### 数据结构

##### Memtable

Memtable 中存储的是位于内存中的有序数据。为了维持数据的有序性以及 Rust 特性，在 WondFS 中采用了由 Rust 标准库提供的 BTreeSet 作为有序数据结构。 

BTreeSet 采用 B-Tree 实现，由于现代 CPU 的快速发展，其指令执行速度较快，相比之下，内存的存取速度与 CPU 之间产生了较大差距，尽管传统的红黑树拥有较好的理论复杂度，但是由于缓存不友好，产生的 cache miss 较多，从而实际使用的性能并不如缓存友好的 B-Tree。

为防止 KV 消耗过多的内存，WondFS 会设定 Memtable 的阈值，当超过阈值时，KV Engine 会遍历 Memtable 中的 BTreeSet，生成 SSTable 文件，并将其写入到 flash 中。

##### SSTable

SSTable 文件内部是有序的键值对，不同的 SSTable 文件的 Key 值区域不重叠（除刚由 Memtable 写入的 SSTable）。当 SSTable 文件过多时，会触发合并操作，会选择两个 SSTable 进行合并，以消除删除操作的重复记录和减少文件数量，从而减少读操作查询的文件数量从而优化读性能。

##### WondFS 适配

由于在文件系统内部只有 Block 的概念，而没有文件的概念，为了适配传统的LSM-Tree的实现，WondFS需要在内部将 Block 包装成文件，以支持 SSTable 文件的组织。

为了支持 SSTable，WondFS在内部包装了 BlockIterator 和 FileIterator，其主要实现了 Block 内部的顺序访问和以文件形式组织的块的顺序访问，并提供了 SSTable 所需要的接口，如 get, next, hasNext 等。

#### 增删查改操作

目前在 WondFS 中还未支持 Write Ahead Log (WAL) 功能，我们将在之后的工作中添加这部分功能，但在设计时保留了这部分的逻辑。

##### 写操作

写入时，需要先将数据写入到 WAL 中以保证数据一致性，随后，数据会被写入到 Memtable 中，在 Memtable 中，只需要使用 BTreeSet 实现的接口插入到内部数据结构中。

若 Memtable 的数据量已经达到或者超过阈值，将会触发 SSTable 生成操作，通过遍历 BTreeSet 生成顺序键值对，调用外部实现的 Block 写入操作，写入到 Flash 中，随后清空 BTreeSet。

若 SSTable 文件数量达到或者超过阈值，会触发 SSTable 合并操作，在目前的 WondFS 操作中，会选择最久的两个 SSTable 文件进行合并，合并操作由归并排序实现。因为这两个文件中存储的数据为访问较少的冷数据，合并成大文件之后对于读性能产生的影响较小。

##### 读操作

读操作会遍历 Memtable，SSTable。其中 Memtable 中的读只需要使用 BTreeSet 提供的接口实现。而在 SSTable 中，需要按照生成时间的先后依次查询所有的 SSTable，直到查找到对应键值对或者查询完毕所有的 SSTable。 由于在 SSTable 中已经是有序的键值对，所以可以使用二分查找或者指数查找快速查询（目前实现了简易的顺序查找，我们会在将来的工作中添加二分查找或者以跳表的形式实现）。

##### 删操作

删操作在 WondFS 中的实现为写入某一特定 Value 以表示该 Key 被删除，当读操作读出该特定值时就表示该 Key 在 KV 中已被删除。

##### 更新操作

更新操作在 WondFS 中只需要以写入新的键值对的形式实现。因为读操作会按照 SSTable 文件的生成的顺序反向查询，所以保证删除操作一定会在其对应的插入操作之前查询到。

#### 实现细节

在 WondFS 中，Key 和 Value 都是以固定长度实现，为 16 Bytes，KV Engine 会使用 flash 上的第 6~9 块 Block 作为空间。
