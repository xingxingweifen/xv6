1. **原因：** 操作系统可以使用页表硬件的技巧之一是延迟分配用户空间堆内存。Xv6应用程序使用sbrk()系统调用向内核请求堆内存。
   1. 内核为一个大请求分配和映射内存可能需要很长时间；
   2. 有些程序申请分配的内存比实际使用的要多（例如实现稀疏数组）；
   3. 延迟分配可以让sbrk()调用运行的更快。
<br>

2. 首先在原来的xv6系统中sbrk系统调用的分配方式是eager alloction，我们要做的就是在分配内存的时候并不实际分配内存。
```c
uint64 sys_sbrk(void)
{
    int addr;
    int n;

    if(argint(0, &n) < 0)
        return -1;
    addr = myproc()->sz;
    if (growproc(n) < 0)
        return -1;
    return addr;
}
//修改过后
uint64 sys_sbrk(void)
{
    int addr;
    int n;

    if(argint(0, &n) < 0)
        return -1;
    addr = myproc()->sz;
    if (n >= 0){
        myproc()->sz += n;
    }else{
        if (growproc(n) < 0)
            return -1;
    }
    return addr;
}
```
3. 之后当用户进程实际使用到未分配的虚拟地址时，会触发xv6的trap机制，进入到usertrap函数当中，我们可以通过获取STVAL寄存器的值获取到出错的虚拟地址，SCAUSE寄存器的值获取到出错的原因（可以判断是读错误还是写错误）。此时就可以真正对包含出错地址的页分配物理空间了。
```c
void usertrap(void){
    ...
    if (r_scause() == 8){
        ...
    }else if (r_scause() == 13 || r_scause() == 15){
        uint64 va = r_stval();//获取出错的虚拟地址
        printf("page fault va = %p\n", va);

        if (va < p->sz && va >= p->trapframe->sp){//判断出错的虚拟地址是否位于堆区
            uint64 mem = (uint64)kalloc();
            if (mem == 0){
                p->killed = 1;
            }else{
                memset((void *)mem, 0, PGSIZE);
                va = PGROUNDDOWN(va);//获取页首地址
                if (mappages(p->pagetable, va, PGSIZE, mem, PTE_R | PTE_W | PTE_U) < 0){
                    kfree((void *)mem);
                    p->killed = 1;
                }
            }
        }else{
            p->killed = 1;
        }        
    }
    ...
}
```
<br>

4. 同时我们需要处理下解除映射的函数uvmunmap，因为之前的xv6中不存在未映射的虚拟内存。
```c
void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
    uint64 a;
    pte_t *pte;

    if((va % PGSIZE) != 0)
        panic("uvmunmap: not aligned");

    for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
        if((pte = walk(pagetable, a, 0)) == 0)//未建立映射关系时会返回0
            continue;
        //   panic("uvmunmap: walk");
        if((*pte & PTE_V) == 0)//最后一级页表未建立映射关系
            continue;
        //   panic("uvmunmap: not mapped");
        if(PTE_FLAGS(*pte) == PTE_V)
            panic("uvmunmap: not a leaf");
        if(do_free){
            uint64 pa = PTE2PA(*pte);
            kfree((void*)pa);
        }
        *pte = 0;
    }
}
```
<br>

5. 这里有个细节，对于未分配的虚拟地址，如果此时调用系统调用如read和write，由于系统调用陷入内核是走的r_scause == 8的if所以并不会给懒分配的虚拟内存地址分配空间，因此这里还需要处理一下。在系统调用时获取用户空间的虚拟地址使用的是argaddr函数，可以在此函数中检查虚拟地址是否分配到物理地址。
```c
int
argaddr(int n, uint64 *ip)
{
    *ip = argraw(n);
    struct proc *p = myproc();

    if (walkaddr(p->pagetable, *ip) == 0){
    //虚拟地址未分配内存
        uint64 va = *ip;
        if (va < p->sz && va >= p->trapframe->sp){
            uint64 mem = (uint64)kalloc();
            if (mem == 0){
                return -1;
            }else{
                memset((void *)mem, 0, PGSIZE);
                va = PGROUNDDOWN(va);
                if (mappages(p->pagetable, va, PGSIZE, mem, PTE_R | PTE_W | PTE_U) < 0){
                    kfree((void *)mem);
                    return -1;
                }
            }
        }else{
            return -1;
        }
    }
    return 0;
}
```