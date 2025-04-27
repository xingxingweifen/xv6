### 增长文件大小
```c
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)

// On-disk inode structure
struct dinode {
    short type;           // File type
    short major;          // Major device number (T_DEVICE only)
    short minor;          // Minor device number (T_DEVICE only)
    short nlink;          // Number of links to inode in file system
    uint size;            // Size of file (bytes)
    uint addrs[NDIRECT+1];   // Data block addresses
};
```
**思路**：改变数据结构和数据索引的逻辑。
### 实现软链接/符号链接
```c
symlink(char *target, char *path)
```
**思路**：在xv6系统中新建一个T_SYMLINK文件类型，symlink需要新分配一个inode，并在数据域存储target。要点就是当open函数打开path需要递归地进行软链接，直到打开一个文件。