# F2FS中垃圾回收优化

## Reducing garbage collection overhead of log-structured file systems with GC journaling

### 【问题描述】
在F2FS垃圾回收后，需要触发一次checkpoint用于更新文件系统状态，才能使得被更新的单元节（Section）才能投入重新使用。否则，系统发生crash后，倘若checkpoint利用到已回收的数据块进行系统恢复，则系统将一直崩溃。此外，checkpoint开销极大，每次垃圾回收都要触发checkpoint，开销巨大。

### 【解决方案】
本文提出一种GC journaling机制用于代替垃圾回收过程中Checkpointing。GCJ使用一个日志空间来记录GC移动的块的所有信息。使用日志，可以保证文件系统的一致性，而无需在系统崩溃时进行检查点。通过消除垃圾收集期间的检查点，进而可以显著降低垃圾收集延迟。如图所示，在迁移victim block时写入Journal entries。
