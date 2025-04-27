### broken_sleep
会导致lost wakeup的原因：
```c
sleep:
    acquire(&p->lock)
    p->state = SLEPPING
    p->chan = chan
    swtch()
wakeup:
    for each p in procs:
        acquire(&p->lock);
        if p->state == SLEEPING and p->chan == chan:
            p->state = RUNNABLE
        release(&p->lock);
int done//全局变量
uartwrite(buf):
    //lock
    while not done:
        sleep(&tx_chan)
    send c
    done = 0
    //unlock 
    //头尾加锁肯定不能工作，因为违背了在swtch中只能持有p->lock的原则，而且当中断程序调用的时候也获取锁，这样就会死锁（自旋锁关闭中断）。
uartintr()://中断处理函数
    lock
    done = 1
    wakeup(&tx_chan)
    unlock
```

进行演变：
```c
uartwrite(buf):
    lock
    while not done:
        unlock
        sleep(&tx_chan)
        lock
    send c
    done = 0
    unlock
```
重点就是在unlock和sleep的程序之间可能发生中断，改变done的状态，进而产生lost wakeup的问题。
### 解决lost wakeup问题的方法就是处理unlock和sleep之间的窗口时间
```c
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.
  if(lk != &p->lock){  //DOC: sleeplock0
    acquire(&p->lock);  //DOC: sleeplock1
    release(lk);
    //获取p->lock之后就算release之后发生了中断，在wakeup的程序中由于获取了p->lock所以对p的状态的判断只能自旋等待。
  }

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}
```
lost wakeup不就是 在进程进入SLEEPING状态之前，先调用了wakeup函数。
这里的效果是由之前定义的一些规则确保的，这些规则包括了：

1. 调用sleep时需要持有condition lock，这样sleep函数才能知道相应的锁。
2. sleep函数只有在获取到进程的锁p->lock之后，才能释放condition lock。
3. wakeup需要同时持有两个锁才能查看进程。