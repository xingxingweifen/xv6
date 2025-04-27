1. 可能某些操作系统上并不需要从user pagetable 切换到 kernel pagetable。
2. 在一些系统调用过程中，一些寄存器不用保存。
3. 对于某些简单的系统调用或许根本就不需要任何stack。