# GC

## 基本设计

由于异地更新的特点，垃圾回收是WondFS中重要的一环。在WondFS中，一共有三种垃圾回收的方式。



另外。在GC设计的过程中，需要着重考虑写入放大的影响。写入放大是闪存和固态硬盘（SSD）中一种不良的影响，即实际写入的物理资料量





## 层次结构



## 主要功能

### 前台GC



### 后台GC

后台GC是在WondFS中的一个常驻线程，每隔固定的时间触发，不同与前台GC，

## 功能实现

### BlockTable

#### 相关数据结构

**BlockInfo**



#### 关键函数





### GCManager

#### 相关数据结构



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
