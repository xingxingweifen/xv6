### Memory Mapped Files
**核心思想**：将完整或者部分文件映射到内存中，这样就可以通过内存地址相关的load或者store指令来操纵文件。
**函数签名**：
```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
//含义：将fd代表的文件从偏移位置offset到offset+length映射到进程的虚拟地址addr处
//prot：prot指示内存是否应映射为可读、可写，以及/或者可执行的；您可以认为prot是PROT_READ或PROT_WRITE或两者兼有。
//flag：flags要么是MAP_SHARED（映射内存的修改应写回文件），要么是MAP_PRIVATE（映射内存的修改不应写回文件）。
//offset：假定为0
int munmap(void *addr, uint length);
munmap(addr, length)应删除指定地址范围内的mmap映射。如果进程修改了内存并将其映射为MAP_SHARED，则应首先将修改写入文件。
```
**VMA**:virtual Memory Area，记录mmap创建的虚拟内存范围的地址、长度、权限、文件等等
```c
struct VMA{
    uint64 addr;//记录虚拟内存的起始地址
    uint64 length;//记录虚拟内存的范围
    int prot;//记录保护权限
    int flags;//记录修改的内存是否写回到原文件
    struct file *f;//记录关联的文件
    int off;//文件的偏移量
};
```
### 主要步骤
1. 在proc的结构体中声明VMA数组，并在初始化进程时将数组置为0。
```c
struct proc{
    ...
    struct VMA vmas[16];        //管理的VMA结构体
};
static struct proc* allocproc(void)
{
  ...

found:
  ...

  memset(p->vmas, 0, sizeof(p->vmas));
  return p;
}
```

2. 惰性的填写页表，利用trap机制延迟进程对内存的读写。惰性保证了大文件的mmap是快速的，并且比物理内存大的文件的mmap是可能的。
```c
uint64
sys_mmap(void){
    //1.获取参数
    uint64 addr;
    int prot, flags, fd, off, len;
    struct file *f = 0;
    if (argaddr(0, &addr) < 0 || argint(1, &len) < 0 || argint(2, &prot) < 0
     || argint(3, &flags) < 0 || argfd(4, &fd, &f) < 0 || argint(5, &off))
        return -1;
    if (f->readable == 0 || (f->writable == 0 && ((flags & MAP_SHARED) && (prot & PROT_WRITE))))//排除文件不可读、文件不可写但是需要共享的情况
        return -1;
    //其中addr可以假定获取的参数为0，off获取的参数也为0
    struct VMA *pVMA = (void *)0;
    //填写VMA结构体，并为进程空间分配虚拟内存地址
    struct proc *p = myproc();
    int i;

    for (i = 0; i < 16; ++i){
        if (p->vmas[i].f == 0){
            break;
        }
    }

    if (i== 16)
        return -1;
    pVMA = &p->vmas[i];
    //记录参数
    pVMA->addr = PGROUNDUP(p->sz);
    p->sz = PGROUNDUP(p->sz) + len;//懒分配空间
    pVMA->length = len;
    pVMA->prot = prot;
    pVMA->flags = flags;
    pVMA->f = f;
    pVMA->off = off;

    //增加f的引用计数，以便在文件关闭时结构体不会消失
    filedup(f);

    return pVMA->addr;
}
```
3. 当进程真正对映射的虚拟内存进行读写时，会产生读写错误，进而进入到usertrap中，usertrap函数捕获这种类型的错误，为虚拟内存发生错误的地方分配一页物理内存，并复制相应的文件页。
```c
void usertrap(void){
    if (r_scause() == 8){
        ...
    }else if((which_dev = devintr()) != 0){
    // ok
    }else if (r_scause() == 13 || r_scause() == 15){
    //分配新的页，并复制相应文件的内容
    uint64 falut_va = r_stval();//产生页面错误的虚拟地址
    char *pa = 0;
    struct VMA *pVMA = 0;
    for (int i = 0; i < 16; ++i){
        pVMA = &p->vmas[i];
        if (falut_va >= pVMA->addr && falut_va < pVMA->addr + pVMA->length)
            break;
        pVMA = 0;
    }
    if (pVMA == 0 || (pa = kalloc()) == 0){
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
        printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());        
        p->killed = 1;
    }else{
        //处理标志位
        int prot = 0;
        if (pVMA->prot & PROT_READ){
            prot |= PTE_R;
        }
        if (pVMA->prot & PROT_WRITE){
            prot |= PTE_W;
        }
        if ((prot & PTE_R) == 0 && r_scause() == 13){
            printf("区域不可读\n");
            p->killed = 1;
        }
        if ((prot & PTE_W) == 0 && r_scause() == 15){
            printf("区域不可写\n");
            p->killed = 1;
        }
        memset(pa, 0, PGSIZE);
        if (mappages(p->pagetable, PGROUNDDOWN(falut_va), PGSIZE, (uint64)pa, prot | PTE_U) != 0 ){
            kfree(pa);
            p->killed = 1;
        }else{
            struct inode *ip = pVMA->f->ip;
            ilock(ip);
            //根据inode从对应的buffer cache中读取一页数据
            if (readi(ip, 0, (uint64)pa, PGROUNDDOWN(falut_va) - pVMA->addr, PGSIZE) == 0){
                panic("usertrap read");
            }
            iunlock(ip);
        }
    }
}
```
4. 实现munmap
```c
uint64 sys_munmap(void){
    uint64 addr;
    int len, i;
    //获取参数
    if (argaddr(0, &addr) < 0 || argint(1, &len) < 0)
        return -1;
    struct proc *p = myproc();
    struct VMA *pVMA = 0;
    //从p->vmas.addr中判断范围，获取对应的VMA成员
    for (i = 0; i < 16; ++i){
        if (p->vmas[i].f == 0)
            continue;
        if (addr >= p->vmas[i].addr && addr < p->vmas[i].addr + p->vmas[i].length){
            pVMA = &p->vmas[i];
            break;
        }
    }
    if (pVMA == 0){
        panic("munmap");
    }

    if ((pVMA->flags & MAP_SHARED) && (pVMA->prot & PROT_WRITE)){
        //将页面写回内存
        filewrite(pVMA->f, addr, len);
    }

    if (addr == pVMA->addr && len == pVMA->length){
        //全部释放
        fileclose(pVMA->f);//减少引用计数
        memset(pVMA, 0, sizeof(struct VMA));
    }else if(addr == pVMA->addr && len < pVMA->length){
        //释放头部部分
        pVMA->addr = addr + len;
        pVMA->length = pVMA->length - len;
    }else if (addr > pVMA->addr){
        //释放尾部部分
        pVMA->length = pVMA->length - len;
    }
    uvmunmap(p->pagetable, addr, len / PGSIZE, 1);
    return 0;
}
```