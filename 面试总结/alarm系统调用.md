alarm系统调用就是充分利用了trap机制。
1. 我们需要首先实现sigalarm系统调用，函数签名为sigalarm(int ticks, void(*handler))，相应地可以在进程有关的结构体中添加ticks和handler和diff成员，分别记录需要报警的时间间隔、处理函数和已经过去的时间。
2. 当用户程序通过sigalarm系统调用进入内核空间时，可以有伪代码：
```c
if (p->handler == 0 && p->ticks == 0){
    p->ticks = ticks;
    p->handler = handler;
    p->diff = 0;
}
```
3. 内核会相应计时器中断，在usertrap中有：
```c
if (which_dev == 2)
```
为了从内核空间返回到用户函数空间时，调用handler函数，可以将保存在p->trampframe->epc修改为handler的地址。所以可以有伪代码如下：
```c
if (which_dev == 2){
    ++p->diff;
    if (p->diff == p->ticks && p->handler != 0){
        p->trapframe->epc = p->handler;
        p->diff = 0;
    }
}
```
4. 直接这么做会导致系统崩溃，因为返回到用户空间后并没有返回到产生中断的地方，为此我采取的解决方案是在handler函数最后调用sigreturn系统调用，函数签名为：
```c
int sigreturn(void)
```
该函数的目的就是为了在handler函数结束调用时，重新进入内核空间，将当前的p->trapframe修改为与上一次中断产生的状态一致。为此，我们需要在进程的结构体中继续声明一个trapframe结构的成员，当p->diff == p->ticks时，此时就要保存p->trapframe进p->trapframe_copy。有伪代码如下：
```c
if (which_dev == 2){
    ++p->diff;
    if (p->diff == p->ticks && p->handler != 0){
        memmove(p->trapframe_copy, p->trapframe, sizeof(p->trapframe));
        p->trapframe->epc = p->handler;
        p->diff = 0;
    }
}
```
5. 因而在sigreturn系统调用中应该将p->trapframe_copy复制给p->trapframe
```c
memmove(p->trapframe, p->trapframe_copy, sizeof(p->trapframe));
p->diff = 0;
```