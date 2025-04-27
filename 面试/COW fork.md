1. **原因**：xv6中的fork()系统调用将父进程的所有用户空间内存复制到子进程中。如果父进程较大，则复制可能需要很长时间。而且一但复制完成后子进程就调用exec()系统调用则会导致子进程丢弃复制的内存。造成极大的浪费。
<br>

2. **解决方案**：使用COW-fork，只为子进程创建一个页表，其指向对应的父进程的物理内存。COW-fork将父子进程的页表PTE都标记为只读状态，一但某个进程需要对物理内存进行修改时，就会产生page fault，这时进入trap后我们捕获这种错误，对出错的虚拟地址分配物理内存并复制相应的物理内存，之后首先将原先的PTE置为可读可写，再将新分配物理内存的PTE的标志位置也为可读可写。
    1. **可能存在的问题**：当一个父进程有多个子进程时，这些子进程可能只映射到一个物理页面，所以当一个子进程退出时不能直接释放物理内存，可以使用引用计数的方法确定何时释放物理页面。
<br>

3. **主要步骤**
    1. 取PTE的保留标志位分配为COW-fork标志位
    ```c
    #define PTE_COW (1L << 8)   //标识未COW fork bit
    ```
    2. 修改uvmcopy()将父进程的物理页映射到子进程，而不是分配新页。在子进程和父进程的PTE中清除PTE_W标志。
    ```c
    int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
    {
        pte_t *pte;
        uint64 pa, i;
        uint flags;
        //   char *mem;

        for(i = 0; i < sz; i += PGSIZE){
            if((pte = walk(old, i, 0)) == 0)
                panic("uvmcopy: pte should exist");
            if((*pte & PTE_V) == 0)
                panic("uvmcopy: page not present");
            pa = PTE2PA(*pte);
            flags = PTE_FLAGS(*pte);
            if (flags & PTE_W){//只有原来的物理内存可读时才进行COW
                flags = (flags | PTE_COW) & (~PTE_W);
                *pte = (*pte & (~PTE_W)) | PTE_COW;
            }
            // if((mem = kalloc()) == 0)
            //   goto err;
            // memmove(mem, (char*)pa, PGSIZE);
            if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
            //   kfree(mem);
                goto err;
            }
            kaddrefcnt((char*)pa);//增加物理内存的引用
        }
        return 0;

        err:
            uvmunmap(new, 0, i / PGSIZE, 1);
            return -1;
    }
    ```
    3. 修改usertrap()以识别页面错误。当COW页面出现页面错误时，使用kalloc()分配一个新页面，并将旧页面复制到新页面，然后将新页面添加到PTE中并设置PTE_W。
    ```c
    //首先为每个物理内存设置引用计数
    struct ref_stru {
        struct spinlock lock;
        int cnt[PHYSTOP / PGSIZE];  // 使用数组记录引用计数
    } ref;
    //释放物理内存时减少引用计数
    void kfree(void *pa)
    {
        struct run *r;

        if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
            panic("kfree");

        // 只有当引用计数为0了才回收空间
        // 否则只是将引用计数减1
        acquire(&ref.lock);
        if(--ref.cnt[(uint64)pa / PGSIZE] == 0) {
            release(&ref.lock);

            r = (struct run*)pa;

            // Fill with junk to catch dangling refs.
            memset(pa, 1, PGSIZE);

            acquire(&kmem.lock);
            r->next = kmem.freelist;
            kmem.freelist = r;
            release(&kmem.lock);
        } else {
            release(&ref.lock);
        }
    }    
    //分配内存时初始化引用计数为1
    void *kalloc(void)
    {
        struct run *r;

        acquire(&kmem.lock);
        r = kmem.freelist;
        if(r) {
            kmem.freelist = r->next;
            acquire(&ref.lock);
            ref.cnt[(uint64)r / PGSIZE] = 1;  // 将引用计数初始化为1
            release(&ref.lock);
        }
        release(&kmem.lock);

        if(r)
            memset((char*)r, 5, PGSIZE); // fill with junk
        return (void*)r;
    }
    //增加物理内存的引用计数
    int kaddrefcnt(void* pa) {
        if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
            return -1;
        acquire(&ref.lock);
        ++ref.cnt[(uint64)pa / PGSIZE];
        release(&ref.lock);
        return 0;
    }    
    ```
    ```c

    //对va进行检查是否是COW类型
    int cowpage(pagetable_t pagetable, uint64 va){
        if (va >= MAXVA){
            return -1;
        }
        pte_t *pte = walk(pagetable, va, 0);
        if (pte == 0){
            return -1;
        }
        if ((*pte & PTE_V) == 0){
            return -1;
        }
        return (*pte & PTE_COW ? 0 : -1);
    }
    //实现COW策略
    //1.若当前虚拟地址va映射的物理内存pa的引用为1，则将PTE的标志位置为可读可写即可。
    //2.若当前pa的引用 > 1，则分配新页，拷贝后建立映射，修改标志位为可读可写。
    void *cowalloc(pagetable_t pagetable, uint64 va){
        if (va % PGSIZE != 0){
            return 0;
        }

        uint64 pa = walkaddr(pagetable, va);//获取对应的物理地址
        if (pa == 0){
            return 0;       
        }

        pte_t *pte = walk(pagetable, va, 0);

        if (krefcnt((char *)pa) == 1){
            //只剩一个进程对此物理地址存在引用
            //则直接修改对应的PTE即可
            *pte |= PTE_W;
            *pte &= ~PTE_COW;
            return (void *)pa;
        }
        //多个进程对物理内存存在引用
        //需要分配新的页面，并拷贝旧页面的内容
        char *mem = kalloc();
        if (mem == 0){
            return 0;
        }

        //复制旧页面内容到新页面
        memmove(mem, (char *)pa, PGSIZE);

        //清除PTE_V否则在mappages中会判定为remap
        *pte &= ~PTE_V;

        //为新页面添加映射
        if (mappages(pagetable, va, PGSIZE, (uint64)mem, (PTE_FLAGS(*pte) | PTE_W) & ~PTE_COW) != 0){
            kfree(mem);
            *pte |= PTE_V;
            return 0;
        }

        kfree((char *)pa);

        return mem;
    }
    void usertrap(void){
        ...
        if (r_scause() == 8){
            ...
        }else if (r_scause() == 15){
            uint64 va = r_stval();//获取出错的虚拟地址
            if (va >= p->sz || cowpage(p->pagetable, va) != 0 || cowalloc(p->pagetable, PGROUNDDOWN(va)) == 0){
                p->killed = 1;
            }            
        }
        ...
    }
    
    ```