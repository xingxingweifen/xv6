PC：程序计数器
mode:表明当前是在supervisor mode还是user mode。
SATP：存放内核页表或者用户页表的物理内存地址
STVEC：指向trampoline page的首地址
SEPC：保存用户程序的程序计数器
SSRATCH寄存器：指向trampframe的首地址
SP：栈指针