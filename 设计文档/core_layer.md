## 功能实现

### SuperStat



### BIT

#### 相关数据结构

**BITSegement**

* used_map
* last_erase_time
* erase_count
* average_age
* reserved

BITSegement是BIT Region中的记录单元，每一个segment包含一个Main Area中block的信息。一个sgement在磁盘上占32字节，16字节128位通过0/1记录block中page的使用情况，0为clean，1为dirty/used。4字节记录block上次擦除时间，4字节记录block的擦除次数，4字节记录块的平均生命周期，4字节为保留字段

**BIT**

* table
* sync
* is_op

BIT是磁盘中的BIT Region在内存中的拷贝，table是哈希表，在内存中建立起block的块号到BITSegment的映射。sync参数记录BIT是否与磁盘中的内容相同，当BIT发生更改，需要将更改同步到磁盘中。is_op用于优化同步BIT的次数，当一个文件的写入过程中，BIT 可能会发生很多次变化，通过is_op参数保证BIT在这种情况下只用和磁盘同步一次，从而避免大量不必要的写入和I/O

#### 核心函数

**init_bit_segment**

CoreManager挂载文件系统第一次读取BIT Region时调用，将读出的segment通过init_bit_segment存入BIT

**set_page**

CoreManager对磁盘的写入、擦除操作会修改page的使用情况，通过set_page方法将修改同步进入BIT

**get_page**

从BIT中读取page的使用情况，true代表page处于dirty/used的状态，false代表page处于clean的状态

**set_last_erase_time**

CoreManager对磁盘的擦除操作会改变块的上次擦除时间，通过这个函数将修改同步进入BIT

**set_erase_count**

CoreManager对磁盘的擦除操作会改变块的擦除次数，通过这个函数将修改同步进入BIT

**encode**

将BIT中的所有信息进行编码，返回能直接写入BIT Region的数据内容

**need_sync**

通过is_op和sync变量判断是否要将BIT同步更新到磁盘中的BIT Region

### PIT

#### 相关数据结构

**PITStrategy**

* Map
* Serial
* None

与BIT Region不同，PIT Region存储的是Main Area在使用中的page信息，即在使用中的page中的数据所属文件的ino。BIT Region需要记录每个block的信息，而PIT Region只用记录部分page的信息，这意味着如果像BIT Region一样在磁盘中顺序写，按照所处的位置作为block号，那么PIT Region在需要记录page数过少时，必然是非常稀疏，会产生不必要的空间浪费。为了解决这个问题，PIT Region对于信息的编码方式采用了两种策略。当在使用中的page数量足够多时，按照32字节为一个单位存储，32字节标识的是ino，而page的address就是这个单位在PIT Region中的顺序写的位置。当page并没有那么多时，PIT Region以64字节为一个单元存储，前32字节是page的address，后32字节是page对应的ino。默认情况下是当超过半数的page在使用中，采用Serial策略，否则采用Map策略。策略的选择通过PIT Region中特殊位置的Magic Number标识。策略的选择和PIT Region的编码解码是一种零成本抽象，对上完全不可见，且没有增加任何性能上的负担。

**PIT**

* page_num
* table
* sync
* is_op

PIT是磁盘中的PIT Region在内存中的拷贝，table是哈希表，在内存中建立起page的address到ino的映射。sync参数记录PIT是否与磁盘中的内容相同，当PIT发生更改，需要将更改同步到磁盘中。is_op用于优化同步PIT的次数，当一个文件的写入过程中，PIT 可能会发生很多次变化，通过is_op参数保证PIT在这种情况下只用和磁盘同步一次，从而避免大量不必要的写入和I/O。page_num是Main Area中的page总数，在将PIT进行编码时会通过page_num和table的大小决定编码策略。

#### 核心函数

**init_page**

CoreManager挂载文件系统第一次读取PIT Region时调用，将读出的page信息通过init_page存入PIT

**set_page**

CoreManager对磁盘的写入操作会修改page的使用情况，通过set_page方法将修改同步进入PIT

**get_page**

从PIT中读取page对应的ino

**delete_page**

删除page，当使用中的某个page变为脏数据时调用

**clean_page**

删除page，当擦除某个page时调用，需要注意的是，clean_page的调用时机是和delete_page不同的。擦除以block为单位，这意味着clean_page不会做任何检查，当page存在在PIT中会将其丢弃，不存在不会进行任何操作。但是delete_page需要判断page是否在PIT中，若存在则丢弃，不存在意味着文件系统认为一个没有在使用中的page之前是处于使用中的状态，这是一种不一致的假设和期望，出现这种情况，文件系统需要进行报错处理。

**encode**

将PIT中的所有信息进行编码，返回能直接写入PIT Region的数据内容

**need_sync**

通过is_op和sync变量判断是否要将PIT同步更新到磁盘中的PIT Region

### Journal



### VAM



