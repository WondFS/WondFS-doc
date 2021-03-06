# GC

## 基本设计

由于异地更新的特点，垃圾回收是WondFS中重要的一环。在WondFS中，一共有三种垃圾回收的方式。将在主要功能部分介绍实现方式。

另外。在GC设计的过程中，需要着重考虑写入放大的影响。写入放大是闪存和固态硬盘（SSD）中一种不良的影响，即实际写入的物理资料量是写入资料量的多倍。

由于闪存存在可重新写入数据前就必须先擦除，而擦除操作的粒度与写入操作相比低得多，执行这些操作就会多次移动（或改写）用户数据和元数据。因此，要改写数据，就需要读取闪存某些已使用的部分，更新他们，并写入到新的位置，如果新位置在之前已经被使用过，还需连同先擦除；由于闪存的这种工作方式，必须擦除改写的闪存部分比新数据实际需要的大的多。此倍增效应会增加请求写入的次数，缩短SSD的寿命，从而减少SSD能可靠运行的时间。增加的写入也会消耗闪存的带宽，此主要降低SSD的随机写入性能。许多因素会影响SSD的写入放大，一些可以由用户来控制，而另一些则是数据写入和SSD使用的直接结果。

另外耗损均衡也会造成写入放大，如果反复地编程和擦写某区块，而其他区块却没有写入，该区块会早于其他的区块而磨损——过早地结束了SSD的寿命。由于这个原因，SSD控制器使用称为耗损均衡的技术，以尽可能均匀的将写入分配到SSD的所有闪存区块上。理想情况下，每一区块都能写入到最大次数，这样他们都能同时失效。不幸的是，耗损均衡操作会要求移动之前写入后就未改变的数据（冷数据），以使频繁变动的数据（热数据）可以写入到冷数据的区块中，让冷数据的区块达到均衡。数据被重定位，而主控却并没有修改他们，这增加了写入放大，而降低了闪存的寿命。关键是要找到最优算法以使两者同时达到最佳化。

TRIM命令是常用的解决写入放大的方法，TRIM是一个SATA命令，使得操作系统可以告诉SSD不再需要哪些之前保存过数据的区块。可能这些文件已被删除，或整个分区已被格式化。若操作系统替换了一个LBA的同时覆写了一个文件时，SSD就能知道可以标记原来的LBA为过时或无效，在垃圾回收的过程中就不用再保留那些块。如果用户或操作系统删除一个文件，（不只是除去他的一部分），通常只会将该文件标记为已删除，而并未真正擦除磁盘上的实际内容。正因如此，SSD不知道可以擦除文件先前占用的LBA，所以在垃圾回收时仍会保留他们。

在有操作系统支持的情况下，TRIM命令解决了这个问题。当永久删除一个文件或格式化硬盘时，当永久删除一个文件或格式化硬盘时，操作系统依据不再包含有效数据的LBA发送TRIM命令。这可告知SSD可以擦除并重新使用哪些使用中的LBA。垃圾回收过程中需要移动的LBA因此而减少。结果是SSD将有更多的空闲空间，同时获得低写入放大及更高的性能。

因为WondFS的设计出发点是直接管理Raw Flash，并内置了FTL类似的实现，因此TRIM命令在WondFS中是多余的，WondFS垃圾回收直接感知数据的有效性。另外，在我们看来，TRIM命令并不是解决垃圾回收的根本方法，而是一种在已经选定需要移动块情况下的最优策略。他只能解决不必要的脏数据的移动，但是垃圾回收时需要并应该移动的数据并没有实质性的减少，不恰当的比喻有点事后诸葛亮的感觉，是一种挽救措施。我们认为解决写入放大问题的出发点不仅仅在于垃圾回收块的选择和TRIM命令类似的挽救措施，而且关乎于写入时数据块的选择，写入数据才是造成垃圾回收低效的根本原因。因此，在WondFS中我们在选择数据写入位置时采用了一种启发式策略，每一个文件都有内置的生命周期期望，可以理解为发生下次修改的时间期望。我们在为这个文件写入新数据时，会在考虑空间利用率的情况下，同时考虑将新数据放在生命周期期望相近的数据块中。在这种启发式策略下，我们尽量使得一个数据块中所有page的数据都有相近的生命周期期望。这意味着，他们有更大的可能性同时或者大部分数据在某一时间节点变为脏数据，抑或是大部分数据在某一时间节点均保持有效性。也就是，我们尽量避免一个数据块一半数据是脏数据，一半数据是有效数据这种情况的出现。这种情况下大量的数据块，会带来更大的垃圾回收开销，增强写入放大。试想一下，如果一个数据块大部分都是脏数据，小部分是有效数据，那么此时垃圾回收时回收的成本很低，写入放大影响小。如果一个数据块大部分都是有效数据，小部分是脏数据，这意味着除了刻意的整体移动冷数据块，磨损均衡，大部分情况下这不会成为垃圾回收的回收对象。这两种情况都是我们期望出现的情况。简单来说，WondFS垃圾回收和解决写入放大的出发点是启发式的写入和直接管理Raw Flash的内置TRIM实现（这里的实现不是说实现了TRIM，而是不用实现TRIM）。

## 主要功能

### 寻找写入位置

根据文件的生命周期，选择相近生命周期且有足够空间的数据块。空白块的初始生命周期不是0，而是指定的默认值。

### 前台GC

当没有空闲空间写入新数据时，我们需要前台GC清理数据块，前台GC需要注意及时性，WondFS在前台GC中会直接选择利用率最低的数据块进行回收。

### 后台GC

后台GC是在WondFS中的一个常驻线程，每隔固定的时间触发。不同于前台GC，后台GC由两种不同的GC方式，分为普通后台GC和冷数据后台GC。普通后台GC综合利用率和块冷热选择块进行回收，冷数据后台GC回收冷块，即为了考虑磨损均衡，将那些长期没有修改过的冷数据擦除移动到新的位置。

## 功能实现

### BlockTable

#### 相关数据结构

**BlockInfo**

* size
* block_no
* reserved_size
* reserved_offset
* erase_count
* last_erase_time
* average_age
* dirty_num
* clean_num
* used_num
* used_map

BlockInfo记录了一个block的使用情况以及各种基本信息。

**BlockTable**

* size
* table

size记录了block个数，table存储BlockInfo。

**PagedUsedStatus**

* Clean
* Dirty
* Busy

Clean表示page是干净的，dirty表示page存储的是脏数据，Busy表示page存储的数据在使用中。

#### 关键函数

**get_block_info**

获取BlockInfo。

**set_block_info**

设置BlockInfo。

**get_page**

获取page的使用情况。

**set_page**

设置page的使用情况。

**erase_block**

擦除block。

**set_erase_count**

设置block的erase count。

**set_last_erase_time**

设置block的last erase time。

**set_average_age**

设置block的average age。

### GCManager

#### 相关数据结构

**GCStrategy**

* Forward
* BackgroundSimple
* BackgroundCold

Forward是前台GC，BackgroundSimple是后台标准GC，BackgroundCold表示后台回收冷块。

**GCEventGroup**

存储GCEvent事件。

**EraseGCEvent**

GC擦除事件。

**MoveGCEvent**

GC移动page事件。

**GCManager**

* need_sync
* hot_blocks
* normal_blocks
* cold_blocks
* block_table

GC模块的主要控制类。

#### 关键函数

**find_next_pos_to_write**

为一次指定大小的写入寻找写入位置。

**new_gc_event**

生成一次GC需要进行的磁盘操作。

执行流程如下：

* 指定GC采用的策略。
* 调用choose_gc_block方法，根据GC策略选取待回收的块。
* 调用generate_gc_group方法，传入待回收的块号，创建新的GCEventGroup。
* 根据block table里的信息取出块中所有仍在使用的page信息。
* 连续且属于同一个文件的page组为一个移动单位，调用find_next_pos_to_write_except方法，为每个移动单位寻找下一个写入位置，为一次移动单位生成MoveGCEvent，插入GCEventGroup。
* 生成EraseGCEvent插入GCEventGroup。
* 用GCEventGroup记录回收该块需要进行的所有操作后返回GCEventGroup。

**get_page**

为CoreManager提供的操作block table的接口。从block table中获取page的使用信息。

**set_page**

为CoreManager提供的操作block table的接口。设置block table的page的使用信息。

**get_block_info**

为CoreManager提供的操作block table的接口。从block table中获取block的使用信息。

**set_block_info**

为CoreManager提供的操作block table的接口。设置block table中block的使用信息。
