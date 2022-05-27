# Fuse Layer

## 基本设计

WondFS-fuse是使用FUSE技术实现的用户态Flash友好文件系统。一个完整的可以工作的FUSE文件系统包括以下三个部分：

* 注册为文件系统并将操作转发到通信信道的内核驱动程序，用户空间进程负责处理这些操作。
* 用户空间库（libfuse），他帮助用户空间进程建立并运行与内核驱动程序的通信。
* 实际处理文件系统操作的用户空间实现。

内核驱动程序由FUSE项目提供。Linux内核里有一个fuse.ko模块，这个模块是公用的，内核的位置也是位于文件系统层。现在的Linux内核一般都已经支持fuse模块。可以运行下面这个命令，如果Linux机器支持fuse模块，并且已经加载完成，那么这行命令就不会报错。反之，如果当前Linux不支持这个内核模块，那么就会报错。

```shell
$ modprobe fuse
```

接下来是libfuse的选择，libfuse有多种语言的实现和仓库。我们最后使用的是Rust的fuser库。通过fuser库，我们构建FUSE文件系统时，可以充分利用Rust类型接口和运行时特性。

fuser库一开始是从Rust的fuse库fork出来的仓库，fuse已经很久没有维护过了，fuser是更活跃的库，fuer不只是提供绑定，他通过对原始FUSE C库的重写，从而充分利用Rust的体系结构。

FUSE的作用在于使得我们实现的WondFS文件系统可以绕开内核代码，方便测试开发。我们实现对具体的设备的操作仍需使用设备驱动提供的接口，这是可以直接读写块设备文件。相当于只把文件系统摘到用户态，用户直接管理块设备空间。

## 功能实现

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

* 根据parent的ino从inode manager中获取parent inode。
* 判断parent inode是否为None。
* 调用directoy中的dir_lookup方法，根据parent inode和文件名，查找对应的ino。
* 判断ino是否为None。
* 根据ino从inode manager中获取inode。
* 返回inode的元数据信息。

**getattr**

给定ino，从inode manager获取inode，返回inode的元信息。

**setattr**

给定ino，和一系列元数据，更新对应inode的元信息。

**create**

给定parent的ino和文件名以及新文件的元数据，在指定路径，创建并打开一个文件。

执行流程：

* 根据parent的ino从inode manager中获取parent inode。
* 判断parent inode是否为None。
* 调用directoy中的dir_lookup方法，根据parent inode和文件名，查找对应的ino。
* 如果ino已经存在，说明文件已经存在，返回错误。
* 获取parent_inode的元信息，修改时间信息。
* 调用inode manager的i_alloc方法，创建一个新的inode。
* 修改inode的元信息。注意这里要将inode的ref_cnt设置为1，因为需要打开该文件。
* 调用as_file_kind方法，根据调用create传递进来的mode参数判断文件类型
* 如果新文件是目录文件，调用directory的dir_link文件，为目录文件写入.和..两个目录项，.是对应自己，..是对应父目录文件。
* 调用directory的dir_link方法，将新文件写入parent inode的目录项中。
* 调用allocate_next_file_handle为新打开的文件创建file handle，并将需要的参数返回。

**mknod**

创建一个文件节点。创建常规文件、字符设备、块设备、fifo或socket节点。现在只支持目录文件和普通文件。函数执行流程与create类似，但是不打开文件，即新文件的ref_cnt为0。

**mkdir**

创建一个目录文件。函数执行流程与create类似，但是不打开文件，即新文件的ref_cnt为0。

**unlink**

给定parent的ino和文件名，删除指定文件。

执行流程：

* 根据parent的ino从inode manager中获取parent inode。
* 判断parent inode是否为None。
* 调用directoy中的dir_lookup方法，根据parent inode和文件名，查找对应的ino。
* 判断ino是否为None。
* 根据ino从inode manager中获取inode。
* 获取parent inode的元信息，修改时间信息。
* 调用directory的dir_unlink方法将该文件从parent inode的目录项中移除。
* 修改inode的元数据，将n_link减1。
* 判断n_link是否为0，若不为0，说明还有其他文件硬链接至该文件，不用将该文件删除。
* 若n_link为0，调用inode的delete方法，删除inode。

**rmdir**

给定parent的ino和目录名，删除指定目录。

函数执行流程与unlink类似，但是rmdir只能删除空目录，即目录文件的目录项只包括自己和父目录，暂不支持递归删除所有内容。

**rename**

重命名文件。

**link**

给定ino和新的parent的ino以及新的文件名，在指定新路径出创建一个硬链接到原文件。

执行流程：

* 根据ino和parent ino取出inode和parent inode，判断是否存在。
* 调用directory的dir_link方法，在parent inode的目录项中写入新文件。
* 修改inode的元信息，将n_link加1。

**open**

打开一个文件。

执行流程：

* 调用inode_manager的i_get方法获取inode
* 将inode的ref_cnt加1。
* 调用inode_manager的i_put方法。
* 调用allocate_next_file_handle方法，为打开后的文件分配file handle并返回。

**opendir**

打开一个目录文件。

函数执行流程与open类似。

**release**

关闭一个文件。

执行流程：

* 调用inode_manager的i_get方法获取inode
* 将inode的ref_cnt减1。

**releasedir**

关闭一个目录文件。

函数执行流程与releasedir类似。

**read**

从指定文件中的文件范围内读取数据。

调用inode的read方法读取数据。

**write**

从指定文件中的文件范围内写入数据。

调用inode的write方法写入数据。

**readdir**

读取目录文件的目录项。

执行流程：

* 根据ino从inode manager中获取inode。
* 调用inode的read_all方法读取目录文件的所有数据。
* 解析数据，将目录文件中的所有目录项解析出来并返回。

**access**

检查文件是否有访问权限。









