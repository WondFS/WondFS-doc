### JFFS

SLFS将Norflash介质划分为多个Segement，并将每个Segement划分为多个Block，将Block作为最小读写单位，并将Segemnt作为垃圾回收的基本单位

SLFS的文件管理与传统Ext类型文件系统文件管理方式类似，都是利用Inode与Imap实现

SLFS还会在Norflash介质上开辟一个固定的检查点域CPR，该域被用于映射介质上的所有Imap Block，而每一个Imap Block又被用于映射介质上的Inode Block，而Inode Block则指向Inode Block或者Data Block

SLFS挂载时，读取CPR中的内容，获取所有Imap Block信息，在内存中构建全局IMAP

SLFS的GC机制依靠线性扫描Summary Block实现。对于每个Segement，SLFS都将在其末尾加入Summary Block，Summary Block中记录了该Segment中所有Block的信息。通过Summary Block，SLFS便可根据上述规则进行过期Block的判断

* 当Block为Data Block <ino, ioff>
* 当Block为Inode Block <ino, NULL>
* 当Block为Imap Block <NULL, segmentno>

SLFS的优点是在异地更新带来的极好的写平衡

SLFS在GC机制上却仅是线性扫描，这会导致一些不必要的擦写

由于SLFS只有一种Inode类型节点，这导致SLFS不支持硬链接的实现

### JFFS2

JFFS2利用层级管理结构，将数据寻址负担全权搬运到了内存之中，这使得JFFS2在读写性能方面都远大于JFFS。JFFS中存在多种类型的节点，每个节点带有一个公用的节点头用于表明该节点的类型。在JFFS2中最为重要的两类节点，他们分别为Inode型节点与Dirent型节点。JFFS2中Inode节点具备标识文件类型、承载数据以及承载目录的功能，而Dirent型节点的存在使得硬链接得以实现

JFFS2不再存在Data Block概念，因此所有数据都是紧跟节点之后，JFFS2称这种由头和数据组成的结构为数据实体Entity

JFFS2不再利用Summary Block进行GC，而是通过在全局维护一个版本号，并且为每个节点开辟一个obsolete变量来实现。每当JFFS2的GC程序发现过期节点的存在，便会首先调整内存中的数据结构，并前往介质上将该节点的obsolete位置为0，表示其已过期

JFFS2的标记过期操作充分考虑了Norflash的擦写特性，当节点没过期时，其obsolete位为1，当过期时，我们将0写入，此时不需要擦出数据块

inode cache Raw info

inode Full DNode Frag Tree结构

通过红黑树结构建起各个数据实体在文件中的位置信息，从而进一步加快了文件读写速率

为了解决GC不必要的擦写，JFFS2在内存中维护了多种链表结构，这些链表的每个节点表示一个可擦除块，对应于Norflash中的每个Sector

* dirty_list
* free_list
* clean_list

与JFFS线性GC机制不同，JFFS2为GC对象的选择赋予了概率。通过这种方式，JFFS2保证了clean_list中的块也会有几率被擦除掉，从而使冷热数据流动，避免出现频繁使用dirty_list中的可擦除块的情况

JFFS2还对数据添加了压缩与解压操作

JFFS2优点众多，但也有一些致命的缺点

* 挂载时间问题
* 磨损平衡的随机性
* 不容乐观的拓展性

### JFFS3

JFFS3的提出主要目的是解决JFFS2的可拓展性问题，也就是解决内存占用问题。事实上，解决内存占用的唯一方法便是将索引结构再次放回Norflash介质上。JFFS3利用B+树在介质上构建起对数据实体的管理，并采用索引数据分离的方法来保证Wandering Tree的问题