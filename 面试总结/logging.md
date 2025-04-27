### 为什么需要logging
![抽象](https://mit-public-courses-cn-translatio.gitbook.io/~gitbook/image?url=https%3A%2F%2F1977542228-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-MHZoT2b_bcLghjAOPsJ%252F-MRhzbAZwhuzp63wWdRE%252F-MRielGcbrHOzPCrxHcO%252Fimage.png%3Falt%3Dmedia%26token%3Df685aafe-7c22-4965-9936-d811b090023d&width=768&dpr=4&quality=100&sign=aa7907e3&sv=1)
例如在执行echo "hi" > x的第一阶段
![](https://mit-public-courses-cn-translatio.gitbook.io/~gitbook/image?url=https%3A%2F%2F1977542228-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-MHZoT2b_bcLghjAOPsJ%252F-MS5avG44oTj9OPuULJh%252F-MSAYdy7tK51ltglbIKH%252Fimage.png%3Falt%3Dmedia%26token%3Df3478cf9-3024-4eb0-9062-979a627398dc&width=400&dpr=3&quality=100&sign=d4ca4c48&sv=1)

1. 首先是分配inode，因为首先写的是block 33

2. 之后inode被初始化，然后又写了一次block 33

3. 之后是写block 46，是将文件x的inode编号写入到x所在目录的inode的data block中

4. 之后是更新root inode，因为文件x创建在根目录，所以需要更新根目录的inode的size字段，以包含这里新创建的文件x

5. 最后再次更新了文件x的inode
**而一旦在中间出现Power fault之后**
![](https://mit-public-courses-cn-translatio.gitbook.io/~gitbook/image?url=https%3A%2F%2F1977542228-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-MHZoT2b_bcLghjAOPsJ%252F-MS5avG44oTj9OPuULJh%252F-MSA_aEiYxlhdm9pmIAB%252Fimage.png%3Falt%3Dmedia%26token%3De5ab4709-abce-4ad2-b3ba-a675389b4239&width=400&dpr=3&quality=100&sign=36f63356&sv=1)

就可能出现这个inode已经被分配出去了但是，文件系统还没有来得及记录下这个文件就断电了，再次启动之后，这个inode已经分配出去了，却又不可见。所以logging方案解决的是实现类似操作的**原子性：**要么全部不执行，要么全部执行完在写入到磁盘中。
### logging的基本思想
当需要修改一个block时，并不是直接对文件系统进行修改，而是将修改记录在log当中。log有一个commit标志位，当重启的时候，文件系统会查看log的commit的记录值，如果是0的话，那么什么也不做。如果大于0的话，我们就知道log中存储的block需要被写入到文件系统中，很明显我们在crash的时候并不一定完成了install log，我们可能是在commit之后，clean log之前crash的。所以这个时候我们需要做的就是reinstall（注，也就是将log中的block再次写入到文件系统），再clean log。
**logging机制利用了一个block的write是原子操作的性质**
在内存中存在logheader和对应的block的复制
![](https://mit-public-courses-cn-translatio.gitbook.io/~gitbook/image?url=https%3A%2F%2F1977542228-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-MHZoT2b_bcLghjAOPsJ%252F-MS_F6nYy0utF738c_Y7%252F-MS_LgeEJdY8hOFkuQBW%252Fimage.png%3Falt%3Dmedia%26token%3D14090c19-6cca-4918-9fdc-b03add2ac1f7&width=768&dpr=4&quality=100&sign=3b9782c9&sv=1)
### logging的具体实现
```c
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n;//代表有效的log block的数量
  int block[LOGSIZE];//每个log block 的实际对应的block编号   LOGSIZE:30
};
struct log {
  struct spinlock lock;
  int start;//log blockno：2存放logheader的blockno 3:log存储数据的blockno
  int size;
  int outstanding; // how many FS sys calls are executing.  使用sys calls的线程数量
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
```
在xv6中有标志着一次修改文件系统的开始和结束标志函数，分别是begin_op()和end_op()函数。
任何一个文件系统调用的begin_op和end_op之间的写操作总是会走到log_write。log_write函数位于log.c文件。
```c
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorbtion
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);//防止因为LRU机制将buffer cache被驱逐
    log.lh.n++;
  }
  release(&log.lock);
}
```
```c
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){//是否正在committing？
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){//并发的进程数量过多
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;//增加进程数
      release(&log.lock);
      break;
    }
  }
}
```
用来真正commit的函数是end_op()函数。
```c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){//最后一个提交的进程
    do_commit = 1;
    log.committing = 1;
  } else {//否则唤醒begin_op()中睡眠的进程
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```
```c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;//clean log
    write_head();    // Erase the transaction from the log
  }
}
```
```c
// Copy modified blocks from cache to log.
static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *to = bread(log.dev, log.start+tail+1); // log block  拷贝到相应的log block的buffer cache中
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    memmove(to->data, from->data, BSIZE);
    bwrite(to);  // write the log  真正写到磁盘当中
    brelse(from);//减少buffer cache的引用
    brelse(to);
  }
}
```
```c
// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
static void
write_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);//真正的commiting，写入头部 commit point
  brelse(buf);
}
```
这里的操作感觉确实有点不对劲，hh，在buffer cache中搜索属于log的block块，将这些block写到对应的buffer cache中，最后将这些buffer cache写入到磁盘。
```c
// Copy committed blocks from log to their home location
static void
install_trans(int recovering)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    if(recovering == 0)
      bunpin(dbuf);//这是由于log_write多出来的一次引用，log_write设置的原因是因为防止buffer cache被驱逐
    brelse(lbuf);
    brelse(dbuf);
  }
}
```
### file system recovering
```c
void
initlog(int dev, struct superblock *sb)
{
  if (sizeof(struct logheader) >= BSIZE)
    panic("initlog: too big logheader");

  initlock(&log.lock, "log");
  log.start = sb->logstart;
  log.size = sb->nlog;
  log.dev = dev;
  recover_from_log();
}
static void
recover_from_log(void)
{
  read_head();
  install_trans(1); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}
```