# 【题目】：KVSSD: Close Integration of LSM Trees and Flash Translation Layer for Write-Efficient KV Store
#### 【会议】：DATE2018
#### 【作者】：Sung-Ming W （国立台湾大学），Kai-Hsiang Lin

## 【摘要】
- 在键值存储中，包含LSM-tree，文件系统，FTL在内的多层软件栈设计会导致叠加写放大。
- 提出一种紧密集成LSM-Tree和FTL功能的KVSSD，利用FTL映射机制实现免拷贝LSM-Tree压缩，并在闪存中直接分配数据实现有效的垃圾回收
- 实现表明：与传统的分层设计相比，KVSSD能够降低88%的写放大，并提高35%的吞吐量

## 【引言】
- KV存储日益作为大规模数据中心的基础组件，提供有效灵活的数据存储访问
- LSM-tree中的compation操作带来第一层写放大
- 文件系统中的块I/O大小与闪存页大小不匹配（4K VS 32KB），写入或更新操作采用的read-modify-write机制会带来第二层放大
- 由于闪存存在的写前擦除特性，FTL中不定时执行垃圾回收将带来第三层放大

## 【总体设计】
- 下图给出了KVSSD的整体构造，其在闪存中实现了一颗LSM-tree，称作NAND-flash-LSM(nLSM)
- nLSMS构造成一颗key范围树，用于代替传统FTL中的L2P映射表，进而维护K2P映射
- nLSMS中的每个节点指向一块用于存储元数据的闪存页，进而将整个闪存划分成元数据区和数据区
- 元数据区中的闪存页存储K2P索引，即key范围及其页表指针，数据区中的闪存页存储KV对数据

![Image](https://user-images.githubusercontent.com/33679152/170648558-adce7a11-f34f-4932-991e-6883bf900db0.png)

## 【设计优点】
- 上层以键值对的方式读写数据，每次只需读取闪存页中的若干个键值对，进而消除了read-modify-write操作 （我这么理解对吗？原文是it allocates flash pages to SSTable to eliminate read-modify-write operations，emm，将闪存页分配到SSTable上？？？）
- 将元数据闪存页重映射到KV对数据页上可以降低SSTable compaction操作 （下面会介绍，就修改映射关系，数据不需要移动）
- 将不同层级的KV对数据映射到树的不同层级上，方便进行冷热分离，做垃圾回收

## 【K2P映射】
- K2P映射的目的是根据目标KV对的key找到物理闪存页地址，然而完全解析K2P的开销极大，仅使用SSD内部的RAM无法存储下相应的映射表
- 本文提出key范围树来完成K2P映射
- key范围树中的节点包含该节点的key值范围[min, max]和该节点指向的元数据闪存页物理地址
-  元数据闪存页中包含每个KV对数据闪存页中的key值范围[min, max]和对应的闪存物理页地址
- 查询过程：给定目标KV对，使用key查询key范围树，依次比对节点范围，找到对应的元数据闪存页，在元数据闪存页中，依次比对节点范围，找到对应的数据闪存页，进而在闪存页中遍历查找到对应的KV对

## 【SSTable分配】
- 每张SSTable对应一个闪存块，大小为4MB
- 垃圾回收的对象是闪存块

## 【重映射Compaction】
- 重映射Compaction就是在SSTable compation过程中并非实际移动数据闪存页，而是修改元数据闪存块中的隐射关系
- 重映射虽然可以大幅度降低写放大，但会由于不同版本的键值对的存在造成了读放大 （This is because if the target key is within the key range of an overlap page, then the overlap page must be examined before the other pages in the same SSTable）
- 通过在SSTable中维护一个计数器，用于标记来自不同层级的KV对数量，从而决定是否启动重隐射Compation

## 【冷热分离和垃圾回收】
- LSM-tree中层级越往下，热度越低，因而在垃圾回收的时候尽可能在同一级KV页中写入数据，进而保证相似寿命的页面分配到相似的闪存块中







