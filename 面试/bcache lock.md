### buffer cache
#### 作用
在xv6中buffer cache的作用是减少应用进程访问磁盘的时间，所以在内存中分配了对应的空间作为block的拷贝。其采用了LRU（least recently used）的思想，使用循环链表作为存储结构。数据元素以blockno作为唯一标识，当需要查找元素时，xv6通过线性遍历的方式进行查找，最近使用过的block总是插入到头部，因此当查找失败时总是优先从尾部“驱逐”出refcnt计数为0的blockno。
```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];//30

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;

struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno; //标识对应的block num
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
  uint timestamp;
};
```
#### 降低锁竞争
1. **思路：**散列blockno，分配固定数量的桶，每个桶有各自的锁。但是这样做棘手的是，当一个桶中的查找失败时，我们需要去别的桶中查找空闲的struct buf，若另一个CPU也可能同样在这个桶中进行相同的操作就可能互相获取对方的锁，最终形成死锁。还有就是需要找到最近最少使用的struct buf。
```c
#define NBUCKET 13
#define HASH(id) ((id) % NBUCKET)
struct HashBucket{
    struct spinlock lock;
    struct buf head;
};

struct{
    struct buf buf[NBUF];
    struct HashBucket buckets[NBUCKET];
} bcache;
```
```c
void binit(void){
    struct buf *b = 0;
    for (int i = 0; i < NBUCKET; ++i){
        //初始化哈希表桶的锁
        initlock(&bcache.buckets[i].lock, "bcache.bucket");
        bcache.buckets[i].head.prev = &bcache.buckets[i].head;
        bcache.buckets[i].head.next = &bcache.buckets[i].head;
    }

    //将所有的buf全部挂载到buckets[0]上
    for (b = bcache.buf; b < bcache.buf + NBUF; ++b){
        b->next = bcache.buckets[0].head.next;//头插法
        b->prev = &bcache.buckets[0].head;
        initsleeplock(&b->lock, "buffer");
        bcache.buckets[0].head.next->prev = b;
        bcache.buckets[0].head.next = b;
    }
}
```
```c
//比较复杂的是获取相应的block的拷贝
static struct buf *bget(uint dev, uint blockno){
    struct buf *b = 0, *tmp = 0;
    uint id = HASH(blockno);//得到哈希的桶号

    acquire(&bcache.buckets[id].lock);
    //在当前桶中找到对应的设备和blockno
    for (b = bcache.buckets[id].head.next; b != &bcache.buckets[id].head; b = b->next){
        if (b->dev == dev && b->blockno == blockno){
            ++b->refcnt;

            acquire(&tickslock);
            b->timestamp = ticks;//更新时间戳
            release(&tickslock);            

            release(&bcache.buckets[id].lock);
            acquiresleep(&b->lock);
            return b;
        }
    }
    b = 0;
    //未找到，遍历全局的桶，找到满足b->refcnt == 0 && 当前桶中b->timestamp最小的buf
    for (int i = id, circle = 0; circle < NBUCKET; i = HASH(i + 1), ++circle){
        if (i != id){
            //不是id桶
            //直接这样做会导致死锁，例如a桶可能获取b桶的锁。
            //另一个CPU可能散列的是b桶，并且正在获取a桶的锁。这样就可能导致死锁
            if (!holding(&bcache.buckets[i].lock))
                acquire(&bcache.buckets[i].lock);
            else
                continue;
        }
        //i代表桶号
        for (tmp = bcache.buckets[i].head.next; tmp != &bcache.buckets[i].head; tmp = tmp->next){
            if (tmp->refcnt == 0 && (b == 0 || b->timestamp > tmp->timestamp)){
                b = tmp;
            }
        }
        //没有办法做到找到全局最后修改的buf，原因在于在我们释放一个桶的锁之后，其数据结构可能改变。我们不能在假设不变
        //所以一次只能做到桶级的最优
        if (b){
            //1.这个buf来自id桶
            //2.else
            if (i != id){
                b->prev->next = b->next;
                b->next->prev = b->prev;
                //偷出来
                release(&bcache.buckets[i].lock);

                //插入到id桶当中
                b->next = bcache.buckets[id].head.next;//头插法
                b->prev = &bcache.buckets[id].head;
                bcache.buckets[id].head.next->prev = b;
                bcache.buckets[id].head.next = b;
            }
                b->dev = dev;
                b->blockno = blockno;
                b->valid = 0;
                b->refcnt = 1;

                acquire(&tickslock);
                b->timestamp = ticks;
                release(&tickslock);

                release(&bcache.buckets[id].lock);
                acquiresleep(&b->lock);
                return b;            
        }else{
            if (i != id){
                release(&bcache.buckets[i].lock);
            }
        }
    }

    panic("bget: no buffers");
}
```
```c
void
brelse(struct buf *b){
    uint id = HASH(b->blockno);
    if(!holdingsleep(&b->lock))
        panic("brelse");

    releasesleep(&b->lock);

    acquire(&bcache.buckets[id].lock);
    --b->refcnt;//这里不在b->lock内是因为，多个CPU同时执行并不会有什么问题，再说外层还有一把锁。
    if (b->refcnt == 0){
        acquire(&tickslock);
        b->timestamp = ticks;//更新时间戳
        release(&tickslock);
    }
    
    release(&bcache.buckets[id].lock);
}
```