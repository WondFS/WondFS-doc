# 【题目】TABLEFS: Enhancing Metadata Efficiency in the Local File System
#### 【会议】ATC' 13
#### 【作者】Kai Ren, Garth Gibson CMU

## 【引言】
- 背景：并行文件系统和网络文件系统主要是为了提升文件系统的吞吐量和大文件读写性能，没有考虑元数据和小文件读写特性。支持小文件读写的KV存储系统应运而生。
- KV存储提供NoSQL接口和庞大的数据缓存。其关键特征是使用LSM-Tree来支持随机的更新、删除和更新，进而不影响查询性能。
- 本文提出应该使用KV存储，即利用LSM-Tree聚合元数据的特性，来优化元数据和小文件的存储。

