# 【题目】Semantic-awareness for next-generation file systems

## 【摘要】 
-  背景：使用分层目录树的现有存储系统无法满足指数级增长的数据集和具有数十亿文件的艾字节级系统中日益复杂的查询的可扩展性和功能要求。
- 贡献：提出一种支持语义感知管理的文件系统SmartStore，利用潜在语义分析方法分析文件之间的语义关系进而将相关文件聚集到相关语义感知组中，并为文件系统扩充诸如范围查询、top-k查询之类的接口，进而减少文件系统查询迟延。

## 【引言】
下一代文件系统的两项关键特征：
- 为大量并发访问提供低延迟服务
- 提供灵活的I/O接口，允许用户执行高级元数据查询，如范围查询、top-k查询

下一代文件系统的关键研究问题：
- 如何有效地从大量数据中提取有用的知识
- 如何管理大量具有多维属性的文件，而文件的属性维度还在增加
- 如何有效、快速地从大型数据集中提取小而相关的子集，以构建准确高效的的数据缓存，从而方便高端和复杂的应用？

可以解决的办法：
- 支持PB级甚至EB存储很简单，但是用户更关注程序的数据行为和结构属性，因此我们应该根据文件元数据的语义相关关系来管理文件
- 传统的基于时间或空间局部性的方法，无法维护满足大规模系统处理密集数据的应用需求，利用语义感知实现的范围查询和top-k查询能够显著提高数据查询性能。

对于用户而言，范围查询可以回答诸如“我如何查询昨天晚上运行时间低于30分钟但数据量大于2.6G的实验数据”，Top-k查询回答问题“如何找出1月份数据在300MB左右的最相关的10个数据”。
对于系统而言，范围查询和Top-K查询可用户优化诸如数据重删、缓存替换与预取等部件性能。
总而言之，传统基于目录树的文件系统仅能靠暴力搜索实现复杂数据查询，亟须利用语义相关性提高文件系统性能。

## 【文章贡献】
- 针对文件系统元数据的去中心话语义感知管理策略:利用语义相关性分析工具（如：潜在语义索引）分析文件之间的语义，构建语义相关感知组。
- 多查询服务: 首次在超大规模分布式文件系统上构建复杂查询，支持点、范围和top-k查询。


## 【系统架构】
![Image](https://user-images.githubusercontent.com/33679152/170817016-691bbbfe-5ca9-47d9-a173-f087ad7e1a16.png)

- 核心想法：利用元数据语义可以指导将高度相关的文件聚合成组，反过来利用聚合组完成对数据的复杂查询。
- 索引单元：包含位置和映射信息
- 存储单元：包含文件元数据
![Image](https://user-images.githubusercontent.com/33679152/170817023-6311aad2-f17b-450e-b249-9cec372c9a3d.png)


三个关键的功能组件：
 1）分组组件：根据LSI语义分析将元数据分类为存储和索引单元
 2）构建组件：在分布式环境中迭代构建语义R树
 3）服务组件：支持在R树和多查询服务进行插入与删除

用户视图：
- 用户将查询随机发送到存储单元
- 所选存储单元通过使用基于联合多播的方法或基于脱机预计的方法来查询相应的R树节点
- 对于单点查询，使用多层布鲁姆过滤器数组
- 对于复杂查询，主单元检查最小边界矩形（MBR）以确定成员从属关系。（具体见R树的查询）

系统视图：
- 语义分组组件通过挖掘元数据之间的语义，将语义强相关的文件放入同一组中。
- 所有组都采用R树管理

查询模式自动匹配 （因为用户的查询属性不可预测，所以无法指导使用什么属性来构建R树）

## 【设计和实现】
1) 语义分组 （详情请参考李航《统计学习方法》中的潜在语义分析章节）
2) 系统重新配置
-插入过程： 分组定位、阈值调整
	通过使用LSI计算插入节点和群组之间的语义相关性，利用两者之间的语义向量；经过阈值判断，决定是否放入群组中，否则转交给临近群组进行准入验证；对于插入新节点的群组，营更新其MBR（最小边界矩形），使其能够覆盖新插入的节点。
   准入阈值是用于均衡存储节点之间负载的重要的参数，决定了语义相关性的程度，语义分组的成员和大小。
- 删除过程：与R树的删除过程类似，涉及到群组语义向量值的重新计算，MBR区域的更新，子节点的合并和父节点的位置调整等等。
![Image](https://user-images.githubusercontent.com/33679152/170817030-187da552-9d7f-43c8-8efc-e91601d77e45.png)

- 范围查询：R树每个节点均包含MBR区域，通过将范围查询请求发送给任何的存储单元，然后由该存储单元广播查询信息给其父亲节点和同级节点，查询开销为O(log N)，N是存储单元个数。
- TopK查询：当存储节点接受到查询请求后，首先检查其父亲节点，即索引节点，用于获得与查询点最密切的目标节点。通过计算所有目标节点与查询节点之间的距离，获取最大距离值MaxD。此外，通过广播查询消息给同级节点，验证是否存在比最大距离值MaxD更小的目标节点。 （Top-k检索的过程和KD-Tree类似）
![Image](https://user-images.githubusercontent.com/33679152/170817033-eadcfd07-832c-4b24-8e9a-6846ad6cacc3.png)
- 点查询 （借助bloom filter来完成，父节点的BF由子节点的BF采用并操作获取）

