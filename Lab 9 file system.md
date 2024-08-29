# Lab 9 file system

文件系统作为操作系统的重要部分，在前面的实验中都没有涉及。因而，在本实验中，我 们会深入改进 xv6 原先的文件系统，从而学习与文件系统相关的一些概念。

首先切换到 fs 分支



## Experiment 1 : Large files 

#### 1. 实验准备

在原始的 xv6 的实现中，其文件系统的部分与原版的 Unix v6 类似，均使用基于 inode 和目录的文件管理方式，但其 inode 仅为两级索引，共有 12 个直接索引块和 1 个间接索引块，间接索引块可以指向 256 个数据块，故而一个文件最多拥有 268 个数据块。我们需要将 xv6 的文件系统进行扩充，使之可以支持更大的文件。

要点：

- 实现二级间接块索引(doubly-indirect block)的 inode 结构: ip->addrs[] 的前 11 个元素是直接块(direct block), 第 12 个元素是一个一级间接块索引(singly-indirect block), 第 13 个元素是一个二级间接块索引(doubly-indirect block).
- 修改 bmap() 函数以适配 double-indirect block

#### 2. 实验步骤

1. 修改 kernel/fs.h 中的直接块号的宏定义 NDIRECT 为 11.根据实验要求, inode 中原本 12 个直接块号被修改为 了 11 个.

   `#define NDIRECT 11 // lab9-1`

2. 修改 inode 相关结构体的块号数组. 具体包括 kernel/fs.h 中的磁盘 inode 结构体 struct dinode的 addrs 字段; 和 kernel/file.h 中的内存 inode 结构体 struct inode 的 addrs 字段. 将二者数组大小设置为 NDIRECT+2, 因为实际 inode 的块号总数没有改变, 但 NDIRECT 减少了 1.

   ```c
   // On-disk inode structure
   struct dinode {
     // ...
     uint addrs[NDIRECT+2];   // Data block addresses  // lab9-1
   };
   
   // in-memory copy of an inode
   struct inode {
     // ...
     uint addrs[NDIRECT+2];    // lab9-1
   };
   ```

3. 在 kernel/fs.h 中添加宏定义 NDOUBLYINDIRECT, 表示二级间接块号的总数, 类比 NINDIRECT. 由于是二级, 因此能够表示的块号应该为一级间接块号 NINDIRECT 的平方.

   ```perl
   #define NDOUBLYINDIRECT (NINDIRECT * NINDIRECT) // lab9-1
   ```

4. 修改 kernel/fs.c 中的 bmap() 函数.该函数用于返回 inode 的相对块号对应的磁盘中的块号.由于 inode 结构中前 NDIRECT 个块号与修改前是一致的, 因此只需要添加对第 NDIRECT 即 13 个块的二级间接索引的处理代码. 处理的方法与处理第 NDIRECT 个块号即一级间接块号的方法是类似的, 只是需要索引两次.

   ```c
   static uint
   bmap(struct inode *ip, uint bn)
   {
     uint addr, *a;
     struct buf *bp;
   
     if(bn < NDIRECT){
       if((addr = ip->addrs[bn]) == 0)
         ip->addrs[bn] = addr = balloc(ip->dev);
       return addr;
     }
     bn -= NDIRECT;
   
     if(bn < NINDIRECT){
       // Load indirect block, allocating if necessary.
       if((addr = ip->addrs[NDIRECT]) == 0)
         ip->addrs[NDIRECT] = addr = balloc(ip->dev);
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       if((addr = a[bn]) == 0){
         a[bn] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       return addr;
     }
   
     // doubly-indirect block - lab9-1
     bn -= NINDIRECT;
     if(bn < NDOUBLYINDIRECT) {
       // get the address of doubly-indirect block
       if((addr = ip->addrs[NDIRECT + 1]) == 0) {
         ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
       }
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       // get the address of singly-indirect block
       if((addr = a[bn / NINDIRECT]) == 0) {
         a[bn / NINDIRECT] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       bn %= NINDIRECT;
       // get the address of direct block
       if((addr = a[bn]) == 0) {
         a[bn] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       return addr;
     }
   
     panic("bmap: out of range");
   }
   ```

5. 修改 kernel/fs.c 中的 itrunc() 函数.该函数用于释放 inode 的数据块.由于添加了二级间接块的结构, 因此也需要添加对该部分的块的释放的代码. 释放的方式同一级间接块号的结构, 只需要两重循环去分别遍历二级间接块以及其中的一级间接块.

   ```c
   void
   itrunc(struct inode *ip)
   {
     int i, j, k;  // lab9-1
     struct buf *bp, *bp2;     // lab9-1
     uint *a, *a2; // lab9-1
   
     for(i = 0; i < NDIRECT; i++){
       if(ip->addrs[i]){
         bfree(ip->dev, ip->addrs[i]);
         ip->addrs[i] = 0;
       }
     }
   
     if(ip->addrs[NDIRECT]){
       bp = bread(ip->dev, ip->addrs[NDIRECT]);
       a = (uint*)bp->data;
       for(j = 0; j < NINDIRECT; j++){
         if(a[j])
           bfree(ip->dev, a[j]);
       }
       brelse(bp);
       bfree(ip->dev, ip->addrs[NDIRECT]);
       ip->addrs[NDIRECT] = 0;
     }
     // free the doubly-indirect block - lab9-1
     if(ip->addrs[NDIRECT + 1]) {
       bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
       a = (uint*)bp->data;
       for(j = 0; j < NINDIRECT; ++j) {
         if(a[j]) {
           bp2 = bread(ip->dev, a[j]);
           a2 = (uint*)bp2->data;
           for(k = 0; k < NINDIRECT; ++k) {
             if(a2[k]) {
               bfree(ip->dev, a2[k]);
             }
           }
           brelse(bp2);
           bfree(ip->dev, a[j]);
           a[j] = 0;
         }
       }
       brelse(bp);
       bfree(ip->dev, ip->addrs[NDIRECT + 1]);
       ip->addrs[NDIRECT + 1] = 0;
     }
   
     ip->size = 0;
     iupdate(ip);
   }
   ```

6. 修改 kernel/fs.h 中的文件最大大小的宏定义 MAXFILE. 由于添加了二级间接块的结构, xv6 支持的文件大小的上限自然增大, 此处要修改为正确的值.

   ```c
   #define MAXFILE (NDIRECT + NINDIRECT + NDOUBLYINDIRECT) // lab9-1
   ```

7. 测试，再次编译并运行 xv6 ，然后运行 bigfile

   ```cmd
   $ bigfile
   ..................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
   wrote 65803 blocks
   bigfile done; ok
   ```

   



## Experiment 2 : Symbolic links

#### 1. 实验准备

在诸多类 Unix 系统中，为了方便文件管理，很多系统提供了符号链接的功能。符号链接（或软链接）是指通过路径名链接的文件；当一个符号链接被打开时，内核会跟随链接指向被引用的文件。符号链接类似于硬链接，但硬链接仅限于指向同一磁盘上的文件，而符号链接可以跨磁盘设备。我们需要实现一个系统调用 symlink( char *target, char *path) 用于创建符号链接。

#### 2. 实验步骤

1. 首先，按照通常的方法在 user/usys.pl 和 user/user.h 中加入该系统调用的入口，还有syscall.h 和 syscall.c 中系统调用函数的信息 然后在 kernel/sysfile.c 中加入空的 sys_symlink 。修改头文件 kernel/stat.h ，加一个文件类型代表软链接：

   ```c
   #define T_SYMLINK 4 // Symbolic link
   ```

2. 然后在头文件 kernel/fcntl.h 加入一个标志位，供 open 系统调用使用：

   ```c
   #define O_NOFOLLOW 0x010
   ```

   在xv6中，#define O_NOFOLLOW 0x010 是用来定义打开文件时的标志位之一。这个标志位表示在打开文件时不要跟随符号链接（symbolic link）。

3. 接着在 Makefile 中加入用户程序 symlinktest ，此时应当能正常通过编译。

4. 这些准备 工作完成后，我们可以真正着手实现 symlink(target, path) 和对应的 open 系统调用。在sys_symlink 中，仿照 sys_mknod 的结构，创建节点并写入软链接的数据：

   ```c
   uint64 sys_symlink(void)
   {
       char target[MAXPATH], path[MAXPATH];
       struct inode *ip;
   
       if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
       {
           return -1;
       }
   
       begin_op();
   
       if ((ip = namei(path)) == 0)
       {
           ip = create(path, T_SYMLINK, 0, 0);
           iunlock(ip);
       }
   
       ilock(ip);
   
       if (writei(ip, 0, (uint64)target, ip->size, MAXPATH) != MAXPATH)
       {
           return -1;
       }
       
       iunlockput(ip);
       end_op();
       
       return 0;
   }
   ```

   1. 声明变量：函数内部声明了两个字符数组变量 target[MAXPATH] 和 path[MAXPATH]，用于存储目标路径和链接路径。还声明了一个指向 struct inode 结构体的指针变量 ip，用于表示符号链接对应的索引节点。
   2. 参数解析：通过调用 argstr 函数从用户传入的参数中读取目标路径和链接路径，并将它们分别存储在 target 和 path 变量中。如果参数解析失败，会返回 -1。
   3. 开始操作：调用 begin_op 函数开始文件系统操作，确保原子性和一致性。
   4. 查找链接路径：通过调用 namei 函数查找链接路径对应的索引节点。如果索引节点不存在，则调用 create 函数创建一个新的符号链接节点，并将其类型设置为 T_SYMLINK，然后释放索引节点的锁。
   5. 锁定索引节点：通过调用 ilock 函数锁定索引节点，以防止其他进程对该节点进行并发修改。
   6. 写入目标路径：通过调用 writei 函数将目标路径 target 写入到符号链接的数据区域中。此处的 0 表示偏移量，表示从数据区域的起始位置开始写入；ip->size 表示链接节点的大小；MAXPATH 表示写入的最大长度。如果写入的长度不等于 MAXPATH，则说明写入失败，返回 -1。
   7. 释放索引节点：通过调用 iunlockput 函数释放索引节点的锁，并将其加入到空闲列表中。
   8. 结束操作：通过调用 end_op 函数结束文件系统操作。

5. 然后修改 sys_open ，使之能够跟随软链接并读取其指向的内容：

   ```c
   uint64
   sys_open(void)
   {
   ......
   	if (ip->type == T_SYMLINK)
   	{
   	    if ((omode & O_NOFOLLOW) == 0)
   	    {
   	        char target[MAXPATH];
   	        int recursive_depth = 0;
   	        while (1)
   	        {
   	            if (recursive_depth >= 10)
   	            {
   	                iunlockput(ip);
   	                end_op();
   	                return -1;
   	            }
   	            if (readi(ip, 0, (uint64)target, ip->size-MAXPATH, MAXPATH) != MAXPATH)
   	            {
   	                return -1;
   	            }
   	            iunlockput(ip);
   	            if ((ip = namei(target)) == 0)
   	            {
   	                end_op();
   	                return -1;
   	            }
   	            ilock(ip);
   	            if (ip->type != T_SYMLINK)
   	            {
   	                break;
   	            }
   	            recursive_depth++;
   	        }
   	    }
   	}
   ......
   }
   ```

   - 如果是符号链接，并且打开模式 omode 中没有设置 O_NOFOLLOW 标志，则进行符号链接的解析和跟踪。
   - 在一个循环中，首先判断递归深度是否超过了10层，如果超过则放弃解析，返回失败。
   - 然后通过调用 readi 函数从符号链接节点的数据区域中读取目标路径 target。
   - 解锁并释放之前的符号链接节点（因为要找新的索引节点）。
   - 再通过调用 namei 函数查找目标路径对应的新的索引节点。
   - 锁定新的索引节点。
   - 判断新的索引节点的类型是否为符号链接，如果不是则跳出循环。
   - 如果是符号链接，则递增递归深度，并继续解析和跟踪下一个符号链接节点。

   这段代码的作用是在打开符号链接文件时，判断是否需要跟踪解析符号链接，如果需要，则进行符号链接的解析，并返回最终的非符号链接文件。

6. 测试，再次编译并运行 xv6 ，然后运行 symlinktest

   ```cmd
   $ symlinktest
   Start: test symlinks
   test symlinks: ok
   Start: test concurrent symlinks
   test concurrent symlinks: ok
   ```



## 实验结果

```cmd
== Test running bigfile ==
$ make qemu-gdb
running bigfile: OK (131.1s)
== Test running symlinktest ==
$ make qemu-gdb
(0.7s)
== Test   symlinktest: symlinks ==
  symlinktest: symlinks: OK
== Test   symlinktest: concurrent symlinks ==
  symlinktest: concurrent symlinks: OK
== Test usertests ==
$ make qemu-gdb
usertests: OK (209.4s)
== Test time ==
time: OK
Score: 100/100
```

