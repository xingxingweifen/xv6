### xv6中的进程切换
首先进程切换例如echo进程切换到ls进程。echo可能由于定时器中断或者等待I/O进入内核执行yeild()函数让出CPU。
    1. xv6首先会将echo的内核线程的部分内核寄存器保存在一个struct context对象中。（大部分是callee reg）
    2. 类似的，因为要切换到ls的内核线程，那么ls程序现在的状态必然是RUNABLE，表明ls程序之前运行了一半。这同时页意味着ls程序的用户空间状态已经保存在了它的trapframe中，更重要的是，ls程序的内核线程对应的内核寄存器也已经保存在对应的context对象中。所以接下来，xv6会恢复ls程序的内核线程的context对象，也就是恢复内核线程的寄存器。
    3. 之后ls会继续在它的内核线程上，完成相应的trap机制。
    4. 恢复ls程序的trapframe的用户进程状态，返回到用户空间的ls程序中。
    5. 恢复执行ls程序。
### xv6中的内核线程切换
xv6中内核线程切换的核心就是switch()函数，switch函数通过保存ra和callee reg实现在调度器线程和内核线程的切换。
```c
void scheduler(void)
{
    struct proc *p;
    struct cpu *c = mycpu();
    
    c->proc = 0;
    for(;;){
        // Avoid deadlock by ensuring that devices can interrupt.
        intr_on();
        
        int nproc = 0;
        for(p = proc; p < &proc[NPROC]; p++) {
            acquire(&p->lock);
            if(p->state != UNUSED) {
                nproc++;
            }
            if(p->state == RUNNABLE) {
                // Switch to chosen process.  It is the process's job
                // to release its lock and then reacquire it
                // before jumping back to us.
                p->state = RUNNING;
                c->proc = p;
                swtch(&c->context, &p->context);//当前保存的ra是下一行指令的地址

                // Process is done running for now.
                // It should have changed its p->state before coming back.
                c->proc = 0;
            }
            release(&p->lock);
        }
        if(nproc <= 2) {   // only init and sh exist
        intr_on();
        asm volatile("wfi");
        }
    }
}    
```
```c
.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret    
```
```c
void sched(void)
{
    int intena;
    struct proc *p = myproc();

    if(!holding(&p->lock))
        panic("sched p->lock");
    if(mycpu()->noff != 1)
        panic("sched locks");
    if(p->state == RUNNING)
        panic("sched running");
    if(intr_get())
        panic("sched interruptible");

    intena = mycpu()->intena;
    swtch(&p->context, &mycpu()->context);//当前保存的ra是下一行指令的地址
    mycpu()->intena = intena;
}    
```
[协程](./uthread.md)