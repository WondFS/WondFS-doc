# Translation Layer

## 基本设计

Translation Layer的设计来源于FTL（Flash Translation Layer），但是在功能和目的上都与FTL有很大的不同。在介绍Translation Layer之前，我们需要先了解FTL。FTL的具体细节和技术调研在WondFS-doc/技术调研/ftl.md中有收录。

在WondFS的设计中，磁盘可以分为这样几大区域，Super Region、BIT Region、PIT Region、Journal Region、KV Region、Main Area、Reserved Region。Translation Layer通过使用Reserved Region来实现功能。

在Reserved Region中的block可以被分为这样几类：

* MappingTable：存储物理地址、逻辑地址的映射关系。
* Signature：存储数据签名。
* Used：被用作坏块映射。
* Unused：未被使用的。
* Unknown：未知。

简单介绍一下每一种block的功能。

因为Flash中的数据块在使用过程中，可能会出现坏块，即数据损毁的情况，当一个正在使用中的数据块损坏了，我们需要将其映射到一个可用的空白块中，这是因为连续的逻辑地址更便于上层的实现，这是坏块管理的一部分。我们需要将逻辑地址和物理地址之间的映射关系持久化到Flash中，在这里，我们会将其写在类型为MappingTable的物理块中。需要着重注意的是，FTL中也有这样一层的映射，但这里的两个映射其实是截然不同的，FTL的映射是为了给上层的传统针对机械硬盘设计的文件系统提供一个模拟块设备的接口。但是Translation Layer的映射只会发生在出现坏块的情况下，也就是正常的块的物理地址和逻辑地址是相同，并不需要持久化存储，这对于存储来说也是一种优化。当Reserved Region中的某一数据块被用于坏块管理，代替原有的旧块时，会被标记为Used类型。类型为Signature的数据块中存储的是Flash写入数据时的编码信息，每次对于一个page的写入，会生成128bit的签名，用于纠错和校验。当挂载文件系统时，会读取这些签名的位置信息，将数据对应的签名位置存储在内存中，从而加速读取数据时的校验。

## 主要功能

### 写入缓存

写入缓存使用文件系统内存缓存文件的写入命令和数据，然后之后再将其推送到持久化存储中。由于应用程序不必等待I/O真实地将其数据写入物理存储中，所以通过写入缓存可以提高系统性能。

由于写入缓存将数据缓存在内存中以提高系统性能，因此副作用就是：如果出现任何断电、系统崩溃或设备故障，就可能会丢失信息。

以Windows为例，Windows 10会为内置驱动器默认启用磁盘写入缓存以提升性能，对移动磁盘、U盘等外部磁盘则是默认禁用磁盘写入缓存，以防数据丢失和实现热插拔。

WondFS默认开启写入缓存，大部分数据的写入都会先写入写入缓存，但同样提供了绕开写缓存的接口函数，可以满足文件系统中的某些功能模块的特定需求直接写入磁盘。

### 坏块管理

坏块是有数据发生错误的区域，可以分为两种情况，可纠正的错误和不可纠正的错误。当发生不可纠正的错误时，就需要为逻辑地址重新映射一个新的物理块。

### 纠错编码

纠错编码算法是传输过程中发生错误后能在接收端自行发现并纠正的码。早期被广泛应用于通信领域，在发送端完成数据编码，接收端完成数据译码，保证数据的可靠传输。NAND Flash作为一种广泛使用的存储介质，容易受到PE次数、数据保存时间、温度和Cell间干扰等因素的影响，数据写入后再读出无法保证绝对的正确性，因此需要ECC算法做数据恢复。合适的编码算法不仅保证了数据的正确性，还可以延长Flash 介质的寿命。

在这里，我们期望设计一种自适应的Flash纠错编码，达到稳定性和性能的平衡。

我们调研了常用的编码算法，最终采用并实现了两种常用的编码方式，并根据Flash介质的使用情况，智能地选择相应的编码方式，从而达到我们预设的期望。在存储数据期间不会验证数据，但是在请求数据时会测试是否有错误。如果需要的话，在检测之后会进行纠错阶段，如果错误超过了纠错能力，就会将读取的数据块标记为坏块。

**CRC32**

CRC的全称是循环冗余校验。CRC检测能力极强，开销小，易于用编码器及检测电路实现。从其检测能力来看，他所不能发现错误的几率仅为0.0047%以下。从性能上和开销上考虑，均远远优于奇偶校验及算数和校验等方式。因此，在数据存储和数据通讯领域，CRC可谓是无处不在。CRC按照编码后的长度可以分为CRC8、CRC16、CRC32等。我们使用的是CRC32，即编码后的长度是32bit。

**ECC**

ECC可以检查读取或传输的数据是否有错误，并在发现错误后立即对其进行纠正。ECC与奇偶校验类似，除了它在检测到错误后立即纠正错误。ECC在数据存储和网络传输硬件领域变得越来越普遍，尤其是随着数据速率和相应错误的增加。

**自适应纠错编码**

CRC32是一个性能极高的校验算法，我们在写入数据时默认采用CRC32的方式在Reserved Region更新校验信息。

切换ECC的条件：

* 坏块比例达到0.02。
* 使用寿命达到2/3。
* 出现坏块时间周期足够小。

当数据错误出现的概率较高时，我们使用需要更大计算量但可以提高稳定性的ECC编码方式。

**可拓展性**

每一个Page 4096KB的写入，我们会为其分配128bit的空间用于存储纠错校验信息。128bit记录了使用的校验算法，写入Page位置以及生成的签名。128bit的空间这意味着我们可以直接增加其他编码算法，具有非常高的可拓展性。

## 功能实现

### WriteCache

#### 相关数据结构

**WriteBuf**

* address
* data

写缓存中缓存的数据项，标识一次写记录。

**WriteCache**

* capacity
* cache
* table
* sync

写缓存的容量默认为32，cache用于存储WriteBuf，table哈希表用于标识一条写记录是否存在，避免过多重复的查找。sync标识写缓存的使用情况，是否需要将写缓存中的数据同步到物理存储中。

#### 关键函数

**write**

在写缓存中插入一条写记录。

**read**

从写缓存中读取一条写记录。

**get_all**

读出写缓存中的所有写记录。

**recall_write**

撤回写缓存中的一条写记录，当发生擦写操作时，需要撤回相关page缓存在写缓存中的写记录，避免不必要的写入。

**need_sync**

返回写缓存使用情况，如果写缓存满了，需要将缓存中的写记录持久化到物理存储中。

**sync**

持久化写缓存中的内容后调用，清空写缓存。

### CheckCenter

#### 相关数据结构

**CheckType**

* Crc32
* Ecc

用于区分Page所使用的纠错校验算法。

**CheckCenter**

提供一系列纠错校验算法的使用和相关使用函数。

#### 关键函数

**check**

输入page的数据，以及对应的signature，返回校验结果，如果校验发现有数据位发生错误，且错误在可接受修正范围内，返回修复后的数据。

**sign**

输入page的数据，以及纠错校验所需要使用的算法，生成对应的128bit signature。

**extract_address**

输入signature，返回这个signature所属page的address。

### TranlsationLayer

#### 相关数据结构

**BlockType**

* MappingTable
* Signature
* Used
* Unused
* Unknown

标志Reserved Region数据块的用途。

**TranslationLayer**

* disk_manager
* write_cache
* used_table
* map_v_table
* sign_block_map
* sign_offset_map
* use_max_block_no
* max_block_no
* table_block_no
* sign_block_no
* sign_block_offset
* write_speed
* read_speed
* block_num
* err_block_num
* last_err_time

TranslationLayer是Translation Layer的主要控制类。disk_manager持有下层的磁盘管理类。write_cache是写入缓存，used_table存储了Reserved Region中已经被使用的块，map_v_table存储了坏块的映射关系，当出现坏块时，将坏块的逻辑地址映射到Reserved Region中一个还未被使用的块中。sign_block_map和sign_block_offset存储了Main Area中page数据对应的签名位置。use_max_block_no是上层能够使用的最大逻辑块号，即上层不允许直接操作Reserved Region，max_block_no是磁盘的最大块号。table_block_no存储的是MappingTable块的位置，sign_block_no和sign_block_offse存储的是当前可写入签名的位置，当一个数据块写入签名写满后，需要寻找一个空白块作为后续的签名存储。Translation Layer直接访问Disk Layer，write speed和read speed记录了Disk Layer的平均读写速率。block_num是磁盘的所有块的数量，err_block_num是坏块的数量。last_err_time记录了上次坏块的出现时间。

#### 关键函数

**init**

初始化Translation Layer，为WondFS建立起Translation Layer的视图。

执行流程：

* 根据use_max_block_no和max_block_no确定Reserved Region的范围。
* 依次读取每个块，并对块进行处理，通过块在特定位置的Magic Number对块的类型进行区分。
* 如果该块是Mapping Table，解析block中的数据，在内存中建立起tlb/plb映射表，统计坏块情况。
* 如果该块是Siganture，解析block中的数据，更新内存中的sign_block_map、sign_offset_map、sign_block_no和sign_block_offset。

**read**

根据block_no读取块数据。

执行流程：

* 根据block_no确定block中page的起始地址和结束地址。
* 判断写缓存中是否有对应page的数据，如果写缓存中有相关page的数据，以写缓存内容为标准。
* 根据map_v_table判断该块是否由于出现坏块被映射成其他地址的块。确定实际需要在磁盘上访问的块号。
* 调用disk_manager方法读取该块数据。
* 根据读取时间，更新read_speed。
* 调用check_block方法检查block中的数据是否出现错误。为每个page的数据都根据他的签名进行校验。
* 返回数据。

**write**

根据page的address，写入相应的数据。

执行流程：

* 插入写缓存。
* 如果写缓存不用flush到磁盘上，直接返回。如果写缓存已满，需要flush到磁盘上，继续执行下面的流程。
* 读出写缓存的所有数据。
* 为写缓存的每个page在Reserved Region的签名块中使用纠错编码算法写入签名。
* 调用transfer方法根据map_v_table确定实际需要在磁盘上写入的位置。
* 将数据写入磁盘。
* 更新write_speed。

**write_block_direct**

绕过写缓存，直接写入一个block的数据。

执行流程：

* 为每个page在Reserved Region的签名块中使用纠错编码算法写入签名。
* 调用transfer方法根据map_v_table确定实际需要在磁盘上写入的位置。
* 将数据写入磁盘。
* 更新write_speed。

**erase**

擦除一个block。

执行流程：

* 如果写缓存中有相关page的数据，清除写缓存相关数据。
* 更新sign_block_map和sign_offset_map，清除签名的映射。
* 调用transfer方法根据map_v_table确定实际需要在磁盘上擦除的块号。
* 调用disk_manager的disk_erase方法擦除相关块。
