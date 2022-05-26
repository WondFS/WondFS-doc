## Fuse

用户空间文件系统是一个面向类Unix计算机操作系统的软件接口，他使得无特权的用户能够无需编辑内核代码而创建自己的文件系统，目前Linux通过内核模块对此进行支持。一些文件系统如ZFS、GlusterFS和lustre使用FUSE实现。

Linux用于支持用户空间文件系统的内核模块名叫FUSE，FUSE一次有时特指Linux下的用户空间文件系统。

文件系统是一个通用操作系统重要的组成部分。传统上操作系统在内核层面上对文件系统提供支持。而通常内核态的代码难以调试，效率较低。

Linxu从2.6.14版本开始通过FUSE模块支持在用户空间实现文件系统。

在用户空间实现文件系统能够大幅提高效率，简化了为操作系统提供新的文件系统的工作量。特别适用于各种虚拟文件系统和网络文件系统。但是，在用户态实现文件系统必然会引入额外的内核态/用户态切换带来的开销，对性能会产生一定的影响。

<img src="/Users/yurunjie/Desktop/WondFS-doc/技术调研/截屏2022-05-26 下午9.45.04.png" alt="截屏2022-05-26 下午9.45.04" style="zoom:50%;" />

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0468604d04f74186ba19f1ef9b4338a8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.image)

FUSE是一个用来实现用户态文件系统的框架，这套FUSE框架包含3个组件：

* 内核模块fuse.ko，用来接收vfs传递下来的IO请求，并且把这个IO封装之后通过管道发送到用户态。
* 用户态lib库libfuse，解析内核态转发出来的协议包，拆解成常规的IO请求。
* mount工具fusermount。

/dev/fuse虚设备文件是内核模块和用户程序的桥梁。

用户态的libfuse库的作用只是FUSE协议解析和封装用的，与具体语言无关。libfuse是用C语言实现的FUSE协议库，另外还有Go语言、Rust语言等多种其他语言实现的libfuse库。