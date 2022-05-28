## Flash

Flash Memory是一种非易失性的存储器

在嵌入式系统中通常用于存放系统、应用和数据等。在PC系统中，则主要用在固态硬盘以及主板BIOS中。另外，绝大部分的U盘、SDCard等移动存储设备也都是使用Flash Memory作为存储介质

**主要特性**

* 需要先擦除再写入
* 块擦除次数有限：擦写均衡、坏块管理
* 读写干扰：由于硬件实现上的物理特性，Flash Memory在进行读写操作时，有可能会导致邻近的其他比特发生位翻转，导致数据异常。这种异常可以通过重新擦除来恢复。Flash Memory应用中通常会使用ECC等算法进行错误检测和数据修正
* 电荷泄漏：长期没有使用，会发生电荷泄漏，导致数据错误。不过这个时间比较长，一般十年左右。此种异常是非永久性的，重新擦除可以恢复

**NOR FLASH和NAND FLASH**

根据硬件上存储原理的不同，Flash Memory主要可以分为NOR Flash和NAND Flash两类

* NAND Flash读取速度与NOR Flash相近
* NAND Flash的写入速度比NOR Flash快很多
* NAND Flash的擦除速度比NOR Flash快很多
* NAND Flash最大擦次数比NOR Flash多
* NOR Flash支持片上执行，可以在上面直接运行代码
* NOR Flash软件驱动比NAND Flash简单
* NOR Flash可以随机按字节读取数据，NAND Flash需要按块进行读取
* 大容量下NAND Flash比NOR Flash成本要低很多，体积也更小

由于Flash擦写速度慢，成本高等特性，NOR Flash主要应用于小容量、内容更新少的场景，例如PC主板BIOS、路由器系统存储等

相比于 NOR Flash，NAND Flash 写入性能好，大容量下成本低