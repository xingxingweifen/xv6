### alarm系统调用充分利用了trap的机制
1. 我们需要首先实现sigalarm系统调用，其函数签名如下。为了实现计时的功能可以在进程有关的结构体中添加相应的成员。
```c
int sigalarm(int ticks, void(*handler)(void))

struct proc{
    int ticks;  //报警周期
    void (*handler)(void);  //处理函数
    int diff;   //已经过去的时间
};
```

2. 当用户程序通过系统调用进入内核空间时，sigalarm系统函数可以有伪代码：
```c
if (p->handler == 0 && p->ticks == 0){
    p->ticks = ticks;
    p->handler = handler;
    p->diff = 0;
}
```

3. 在xv6内核中会相应计时器中断，为了从内核空间返回到用户空间时，调用handler函数，可以将保存在p->trapframe->epc修改为handler的地址。
```c
if (which_dev == 2){
    ++p->diff;
    if (p->diff == p->ticks && p->handler != 0){
        p->trapframe->epc = p->handler;
        p->diff = 0;
    }
}
```
4. 当然直接这么做会使系统崩溃，从内核返回后也没有回到中断发生的地方。因此我采取的解决方案是在handler函数最后调用sigreturn系统调用。引入该函数的目的就是为了在handler函数结束调用时，重新进入内核空间，将当前的p->trapframe修改为调用handler函数前的状态。为此，我们需要在进程的结构体中继续声明一个trapframe类型的成员，当p->diff == p->ticks时，保存下p->trapframe。
```c
int sigreturn(void);

struct proc{
    ...
    struct trapframe *trapframe_copy;
};

if (which_dev == 2){
    ++p->diff;
    if (p->diff == p->ticks && p->handler != 0){
        memmove(p->trapframe_copy, p->trapframe, sizeof(p->trapframe));
        p->trapframe->epc = p->handler;
        p->diff = 0;
    }
}
```
5. 在sigreturn系统函数中就应该将p->trapframe_copy复制给p->trapframe
```c
memmove(p->trapframe, p->trapframe_copy, sizeof(p->trapframe));
p->trapframe_copy = 0;
p->diff = 0;//细节
```
