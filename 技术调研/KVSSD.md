# 【题目】：KVSSD: Close Integration of LSM Trees and Flash Translation Layer for Write-Efficient KV Store
#### 【会议】：DATE2018
#### 【作者】：Sung-Ming W （国立台湾大学），Kai-Hsiang Lin

## 【摘要】
- 在键值存储中，包含LSM-tree，文件系统，FTL在内的多层软件栈设计会导致叠加写放大。
- 提出一种紧密集成LSM-Tree和FTL功能的KVSSD，利用FTL映射机制实现免拷贝LSM-Tree压缩，并在闪存中直接分配数据实现有效的垃圾回收
- 实现表明：与传统的分层设计相比，KVSSD能够降低88%的写放大，并提高35%的吞吐量



