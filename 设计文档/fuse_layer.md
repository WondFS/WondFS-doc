### WondFS

**工具函数**



**核心函数**

**new**

初始化文件系统，挂载文件系统。

**allocate_next_file_handle**

分配下一个文件句柄。

**init**

在挂载文件系统后，fuse会自动调用该方法进行一些初始化操作。在这里，如果不存在根目录，自动创建一个根目录文件。

**lookup**

给定parent的ino和文件名，寻找parent目录文件中是否包含该文件。

执行流程：

* 根据parent的ino从inode manager中获取parent inode
* 判断parent inode是否为None
* 调用directoy中的dir_lookup方法，根据parent inode和文件名，查找对应的ino
* 判断ino是否为None
* 根据ino从inode manager中获取inode
* 返回inode的元数据信息

**getattr**

给定ino，从inode manager获取inode，返回inode的元信息

**setattr**

给定ino，和一系列元数据，更新对应inode的元信息

**mknod**



