# disk layer

MTD将 Nand Flash,nor flash 和其他类型的 flash 等设备,统一抽象成MTD 设备来管理,根据这些设备的特点,上层实现了常见的操作函数封装。通过调用linux ioctl对mtd进行io管理。

## 相关数据结构

mtd_oob_buf

* start: u32,

* length: u32,

* ptr:CString


erase_info_user

* start: u32,

* length: u32


mtd_info_user 

* tp: u8,

* flags: u32,

* size: u32, 

* erasesize: u32,

* writesize: u32,

* oobsize: u32, 

* padding: u64  


### MEMGETINFO

获取mtd相关信息，包括设备类型，大小，读写缓冲大小，擦写大小等

#### MEMERASE

对MTD执行擦除操作

#### MEMWRITEOOB

对MTD执行写操作

#### MEMREADOOB

对MTD执行读操作
