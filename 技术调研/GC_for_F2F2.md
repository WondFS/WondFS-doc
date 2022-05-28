# F2FS中垃圾回收优化

## Reducing garbage collection overhead of log-structured file systems with GC journaling

### 【问题描述】
在F2FS垃圾回收后，需要触发一次checkpoint用于更新文件系统状态，才能使得被更新的单元节（Section）才能投入重新使用。否则，系统发生crash后，倘若checkpoint利用到已回收的数据块进行系统恢复，则系统将一直崩溃。此外，checkpoint开销极大，每次垃圾回收都要触发checkpoint，开销巨大。

### 【解决方案】
本文提出一种GC journaling机制用于代替垃圾回收过程中Checkpointing。GCJ使用一个日志空间来记录GC移动的块的所有信息。使用日志，可以保证文件系统的一致性，而无需在系统崩溃时进行检查点。通过消除垃圾收集期间的检查点，进而可以显著降低垃圾收集延迟。如图所示，在迁移victim block时写入Journal entries。

![image](https://user-images.githubusercontent.com/33679152/170817892-74aab184-7554-46b3-8b4d-1d4e9f5444eb.png)

## Reinforcement Learning based Background Segment Cleaning for Log-structured File System on Mobile Devices

### 【摘要】
随着移动设备采用日志结构文件系统，后台段清理对系统性能和存储寿命的影响变得十分显著。 激进的后台段清理方案会产生过多的块迁移，损害NAND存储设备的耐用性，而常规的懒惰策略无法为后续的I/O请求回收足够的段，从而导致前台段清理的发生并延长I/O延迟。 在本文中，提出了一种基于强化学习的方法来平衡权衡，通过学习 I/O 工作负载的行为和逻辑地址空间的状态，所提出的方法可以自适应地将前台段清理的频率平均降低 68.57%，并且比现有方法减少 71.10% 的块迁移次数。

### 【背景】

