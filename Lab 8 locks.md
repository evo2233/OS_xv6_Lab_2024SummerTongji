# Lab 8 locks

前面的很多实验中都涉及到了加锁的问题。锁作为一种互斥机制，对于开发者而言实现简单，理解其行为也不复杂，且能提供可靠的互斥保障。但过多的加锁操作会显著地降低系统的并行性，从而使得多处理机系统无法发挥其应有的性能。本部分实验中，主要涉及对于加锁机制的优化，从而增加 xv6 内核的并行性。

首先切换到 lock 分支



## Experiment 1 : Memory allocator

#### 1. 实验准备

第一部分涉及到内存分配的代码，xv6 将空闲的物理内存 kmem 组织成一个空闲链表 kmem.freelist，同时用一个锁 kmem.lock 保护 ，所有对 kmem.freelist 的访问都需要先取得锁，所以会产生很多竞争。解决方案也很直观，给每个 CPU 单独开一个 freelist 和对应的 lock，这样只有同一个 CPU 上的进程同时获取对应锁才会产生竞争。

需要注意的是，每个 CPU 的 freelist 是从 kmem 的全局 freelist 中分离出来的，因此每个 CPU 仍然可以访问所有的物理内存。只是在分配内存时，系统会优先考虑当前 CPU 的 freelist，从而避免多个 CPU 之间的竞争，提高内存分配的效率。

从另一个角度看，这里为每个 CPU 设置一个内存分配器属于资源的重复设置。通过重复设置资源，可以使得进程直接获得锁而无需等待的概率大大提高，从而降低了进程等待时间的期望值，故整个系统的效率能够得到提高。但是，资源重复设置可能会引起诸如数据一致性等问题，故而会大大增加软件的复杂度

#### 2. 实验步骤

1. 我们首先修改 kalloc.c 中的数据结构，使单一的空闲链表变为多个空闲链表的数组：

   ```c
   struct {
     struct spinlock lock;
     struct run *freelist;
   } kmem[NCPU];
   ```

   同时得修改对应的 kinit 和 kfree 的代码以适应数据结构的修改

2. kinit 需要初始化每个 CPU 的空闲链表和锁：

   ```c
   void
   kinit()
   {
     char buf[10];
     for (int i = 0; i < NCPU; i++)
     {
       snprintf(buf, 10, "kmem_CPU%d", i);
       initlock(&kmem[i].lock, buf);
     }
     freerange(end, (void*)PHYSTOP);
   }
   ```

   具体分析如下：

   1. 首先定义一个 char 类型的数组 buf，用于存储每个 CPU 对应的锁的名称。
   2. 然后使用 for 循环为每个 CPU 分配一个独立的 freelist 和对应的锁。循环的次数是 NCPU，即 CPU 的数量。
   3. 在循环中，使用 snprintf 函数将 CPU 的编号与字符串 "kmem_CPU" 连接起来，生成一个字符串 buf，用于作为锁的名称。
   4. 然后调用 initlock 函数，为当前 CPU 的 freelist 分配一个锁，并将锁的名称设置为 buf。initlock 函数是 xv6 中的一个锁初始化函数，用于初始化一个互斥锁。
   5. 循环结束后，调用 freerange 函数，将空闲的物理内存块添加到全局 freelist 中。该函数用于将一段物理内存块添加到 freelist 中，以便后续的内存分配操作可以从 freelist 中获取空闲的物理内存块。

3. 对分配内存的 kalloc() 修改完成后，我们还需要修改对应的用于释放内存的 kfree() ， 直接将页面加入到当前的 CPU 空闲链表中，实现如下：

   ```c
   void
   kfree(void *pa)
   {
     struct run *r;
   
     if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
       panic("kfree");
   
     // Fill with junk to catch dangling refs.
     memset(pa, 1, PGSIZE);
   
     r = (struct run*)pa;
   
     push_off();
     int cpu = cpuid();
     pop_off();
     acquire(&kmem[cpu].lock);
     r->next = kmem[cpu].freelist;
     kmem[cpu].freelist = r;
     release(&kmem[cpu].lock);
   }
   ```

   下面是对这段代码的详解：

   1. 首先检查参数 pa 是否合法。如果 pa 不是 PGSIZE 的整数倍，说明它不是一个完整的物理页框；如果 pa 的地址小于 end，说明它不在内核代码段之外；如果 pa 的地址大于等于 PHYSTOP，说明它超出了物理内存的范围。如果 pa 不合法，就调用 panic 函数进行内核崩溃。
   2. 然后将物理页框 pa 按字节填充为 1，即使用 memset 函数将 pa 的所有字节都填充为 1。这样做的目的是为了捕获悬空引用，即在释放物理页框后，如果有指针仍然指向该物理页框，就会引用到填充的 1，从而触发错误。
   3. 将物理页框 pa 强制转换为一个 run 结构体指针 r。run 结构体用于表示一个空闲的物理页框，包含一个指向下一个空闲页框的指针 next。
   4. 然后关闭中断，并获取当前 CPU 的编号 cpu。为了确保获取到的 CPU ID 是准确的，我们需要在执行 cpuid() 函数期间禁用中断。这是因为如果在执行 cpuid() 函数的过程中发生中断，可能会导致 CPU 切换到另一个处理器上去执行中断服务程序，而此时获取到的 CPU ID 就不再准确了。
   5. 获取当前 CPU 的 freelist 的锁，并将 r 添加到 freelist 的头部。具体来说，将 r 的 next 指针指向当前 CPU 的 freelist，然后将当前 CPU 的 freelist 指向 r。
   6. 最后释放当前 CPU 的 freelist 的锁。

4. 另外，一个相对麻烦的问题是当一个 CPU 的 freelist 为空时，需要向其他 CPU 的 freelist“借”空闲块。进一步修改 kalloc 函数如下：

   ```c
   void *
   kalloc(void)
   {
     struct run *r;
   
     push_off();
     int cpu = cpuid();
     pop_off();
   
     acquire(&kmem[cpu].lock);
     r = kmem[cpu].freelist;
     if(r)
       kmem[cpu].freelist = r->next;
     else // steal page from other CPU
     {
       struct run* tmp;
       for (int i = 0; i < NCPU; ++i)
       {
         if (i == cpu) continue;
         acquire(&kmem[i].lock);
         tmp = kmem[i].freelist;
         if (tmp == 0) {
           release(&kmem[i].lock);
           continue;
         } else {
           for (int j = 0; j < 1024; j++) {
             // steal 1024 pages
             if (tmp->next)
               tmp = tmp->next;
             else
               break;
           }
           kmem[cpu].freelist = kmem[i].freelist;
           kmem[i].freelist = tmp->next;
           tmp->next = 0;
           release(&kmem[i].lock);
           break;
         }
       }
       r = kmem[cpu].freelist;
       if (r)
         kmem[cpu].freelist = r->next;
     }
     release(&kmem[cpu].lock);
   
     if(r)
       memset((char*)r, 5, PGSIZE); // fill with junk
     return (void*)r;
   }
   ```

   下面是对这段代码的详解：

   1. 首先关闭中断，并获取当前 CPU 的编号 cpu，再开启中断
   2. 获取当前 CPU 的 freelist 的锁，并将 freelist 的头部赋值给一个 run 结构体指针 r。
   3. 如果 freelist 中有空闲的物理页框，则将 freelist 的头部指向 r 的下一个元素。否则，就需要从其他 CPU 的 freelist 中“偷走”一些物理页框。
   4. 在偷取物理页框之前，先释放当前 CPU 的 freelist 的锁。这样可以避免在偷取物理页框时一直占用锁，从而阻塞其他 CPU 的内存分配操作。
   5. 在其他 CPU 中查找空闲的物理页框，直到找到一个非空的 freelist。具体来说，使用 for 循环遍历所有的 CPU，如果当前 CPU 的编号等于 cpu，则跳过；否则，获取当前 CPU 的 freelist 的锁，并将 freelist 赋值给一个 run 结构体指针 tmp。
   6. 如果 tmp 为空，则说明当前 CPU 的 freelist 中没有空闲的物理页框，需要继续查找其他 CPU。如果 tmp 非空，则从 tmp 开始向后遍历 1024 个物理页框，并将它们从其他 CPU 的 freelist 中“偷走”，并添加到当前 CPU 的 freelist 中。偷取物理页框的过程中，需要注意更新 freelist 的头部和尾部。
   7. 偷取完物理页框后，释放其他 CPU 的 freelist 的锁，并打破 for 循环。
   8. 获取当前 CPU 的 freelist 的锁，并将 freelist 的头部赋值给 r。如果 r 非空，则将物理页框的内容按字节填充为 5，即使用 memset 函数将物理页框的所有字节都填充为 5。
   9. 最后释放当前 CPU 的 freelist 的锁。

5. 测试，再次编译并运行 xv6 ，然后运行 kalloctest

   ```cmd
   $ kalloctest
   start test1
   test1 results:
   --- lock kmem/bcache stats
   lock: bcache: #test-and-set 0 #acquire() 13
   lock: bcache: #test-and-set 0 #acquire() 24
   lock: bcache: #test-and-set 0 #acquire() 18
   lock: bcache: #test-and-set 0 #acquire() 16
   lock: bcache: #test-and-set 0 #acquire() 12
   lock: bcache: #test-and-set 0 #acquire() 11
   lock: bcache: #test-and-set 0 #acquire() 16
   lock: bcache: #test-and-set 0 #acquire() 292
   lock: bcache: #test-and-set 0 #acquire() 20
   lock: bcache: #test-and-set 0 #acquire() 12
   lock: bcache: #test-and-set 0 #acquire() 12
   lock: bcache: #test-and-set 0 #acquire() 12
   lock: bcache: #test-and-set 0 #acquire() 20
   --- top 5 contended locks:
   lock: proc: #test-and-set 167169 #acquire() 1534152
   lock: proc: #test-and-set 158184 #acquire() 1534154
   lock: proc: #test-and-set 145193 #acquire() 1534189
   lock: proc: #test-and-set 145058 #acquire() 1534154
   lock: proc: #test-and-set 143800 #acquire() 1534154
   tot= 0
   test1 OK
   start test2
   total free number of pages: 32496 (out of 32768)
   .....
   test2 OK
   ```





## Experiment 2 : Buffer cache

#### 1. 实验准备

Buffer cache 是 xv6 的文件系统中的数据结构，用来缓存部分磁盘的数据块，以减少耗时的磁盘读写操作。但这也意味着 buffer cache 的数据结构是所有进程共享的（不同 CPU 上的也是如此），如果只用一个锁 bcache.lock 保证对其修改的原子性的话，势必会造成很多的竞争。

解决策略是根据数据块的 blocknumber 将其保存进一个哈希表，而哈希表的每个 bucket 都有一个相应的锁来保护，这样竞争只会发生在两个进程同时访问同一个 bucket 内的 block。

#### 2. 实验步骤

1. bcache 的数据结构如下，我们首先改造 bcache 的数据结构，将单个锁改成多个锁，并将缓存块分组：

   ```c
   struct {
   	struct spinlock lock[NBUCKET];
   	struct buf buf[NBUF];
   	struct buf head[NBUCKET];
   } bcache;
   ```

   以上数据结构描述了一个缓存系统，其中包含了 NBUCKET 个桶和 NBUF 个缓存块。每个桶中包含了一个链表，用于存储缓存块。下面是各个字段的含义：

   - struct spinlock lock[NBUCKET]：每个桶的自旋锁数组，用于保护每个桶的访问。
   - struct buf buf[NBUF]：缓存块数组，用于存储缓存数据。
   - struct buf head[NBUCKET]：每个桶的头部，是一个大小为 NBUCKET 的缓存链表数组，每个链表都包含了一个缓存块的头部。

2. 然后在 buf.h 中加上

   ```perl
   #define NBUCKET 13
   #define HASH(x) ((x) % NBUCKET
   ```

   这个定义定义了两个宏，分别是NBUCKET和HASH(x)。

   NBUCKET宏定义了哈希表中桶的数量。在哈希表中，数据项通常被分配到桶中，桶的数量可以影响哈希表的性能。通常情况下，桶的数量应该是一个质数，以降低哈希冲突的概率。这个宏定义中，NBUCKET被设置为13，表示哈希表中有13个桶。

   HASH(x)宏定义用于计算一个整数x的哈希值。哈希值可以用于将数据项分配到哈希表中的桶中，以便快速地查找和访问数据项。在这个宏定义中，使用取模运算将x映射到哈希表中的桶中，具体来说，HASH(x)的计算方式是将x对NBUCKET取模，得到的余数就是x在哈希表中所属的桶的索引值。

3. 在 buf.h 中添加 time 字段，表示最近最少使用的计时器

   ```c
   struct buf {
   	int valid; // has data been read from disk?
   	int disk; // does disk "own" buf?
   	uint dev;
   	uint blockno;
   	struct sleeplock lock;
   	uint refcnt;
   	struct buf *prev; // LRU cache list
   	struct buf *next;
   	uchar data[BSIZE];
   	uint time;
   };
   ```

4. 然后，修改对应的 bcache 初始化函数 binit() ，使之与修改后的数据结构相适应：

   ```c
   void
   binit(void)
   {
     struct buf *b;
   
     for (int i=0;i<NBUCKET;i++)
     {
       initlock(&bcache.lock[i], "bcache");
     }
     // Create linked list of buffers
     // bcache.head[0].prev = &bcache.head[0];
     bcache.head[0].next = &bcache.buf[0];
     for(b = bcache.buf; b < bcache.buf+NBUF-1; b++){
       b->next = b+1;
       initsleeplock(&b->lock, "buffer");
     }
     initsleeplock(&b->lock, "buffer");
   }
   ```

   具体来说，函数执行以下步骤：

   1. 使用initlock函数初始化哈希表中每个桶的锁。initlock函数是用于初始化互斥锁的函数，它将每个桶的锁初始化为一个可用的互斥锁。
   2. 初始化缓存块的链表。将第一个缓存块的指针bcache.buf赋值给bcache.head[0].next，表示链表中的第一个元素是bcache.buf指向的缓存块。然后，使用一个循环将每个缓存块的指针加入到链表中。具体来说，每个缓存块的next指针指向下一个缓存块，最后一个缓存块的next指针指向空指针，表示链表的末尾。
   3. 使用initsleeplock函数为每个缓存块的锁初始化一个睡眠锁。睡眠锁是一种可重入的锁，它可以防止多个线程同时访问同一个缓存块，以避免竞争条件和死锁等问题。

   总之，该函数完成了初始化缓存块子系统的任务，包括初始化哈希表中每个桶的锁、初始化缓存块的链表和为每个缓存块的锁初始化一个睡眠锁。它为后续的缓存块操作提供了必要的基础和资源管理。然后这是把全部空闲区都链接在一个桶子里了，随着之后 bget 函数的不断调用会逐渐均匀分配（别的桶会过来拿）

5. 我们再写一个 write_cache() 函数：

   ```c
   void
   write_cache(struct buf *take_buf, uint dev, uint blockno)
   {
     take_buf->dev = dev;
     take_buf->blockno = blockno;
     take_buf->valid = 0;
     take_buf->refcnt = 1;
     take_buf->time = ticks;
   }
   ```

   首先，该函数的参数take_buf表示选中的空闲块的指针，dev表示指定设备的编号，blockno表示指定的块号。接下来，依次对选中的空闲块的属性进行初始化设置。具体来说：

   - take_buf->dev = dev：将选中的空闲块的设备编号设置为指定设备的编号；
   - take_buf->blockno = blockno：将选中的空闲块的块号设置为指定块号；
   - take_buf->valid = 0：将选中的空闲块的有效标志位清零，表示该块中的数据已经过期；
   - take_buf->refcnt = 1：将选中的空闲块的引用计数设置为1，表示该块目前被引用了一次；
   - take_buf->time = ticks：将选中的空闲块的时间戳更新为当前的时钟滴答数，表示该块最近被使用的时间。

   该函数主要用于初始化选中的空闲块，并标记为指定设备和块号的数据块，为后续的缓存管理操作提供基础。

6. 然后改造 bget() ，按照上面提到的逻辑，将原来的集中管理改为分桶进行管理：

   ```c
   static struct buf*
   bget(uint dev, uint blockno)
   {
     struct buf *b, *last;
     struct buf *take_buf = 0;
     int id = HASH(blockno);
     acquire(&(bcache.lock[id]));
   
     // 在本池子中寻找是否已缓存，同时寻找空闲块，并记录链表最后一个节点便于待会插入新节点使用
     b = bcache.head[id].next;
     last = &(bcache.head[id]);
     for(; b; b = b->next, last = last->next)
     {
   
       if(b->dev == dev && b->blockno == blockno)
       {
         b->time = ticks;
         b->refcnt++;
         release(&(bcache.lock[id]));
         acquiresleep(&b->lock);
         return b;
       }
       if(b->refcnt == 0)
       {
         take_buf = b;
       }
     }
   
     //如果没缓存并且在本池子有空闲块，则使用它
     if(take_buf)
     {
       write_cache(take_buf, dev, blockno);
       release(&(bcache.lock[id]));
       acquiresleep(&(take_buf->lock));
       return take_buf;
     }
   
     // 到其他池子寻找最久未使用的空闲块
     int lock_num = -1;
   
     uint64 time = __UINT64_MAX__;
     struct buf *tmp;
     struct buf *last_take = 0;
     for(int i = 0; i < NBUCKET; ++i)
     {
   
       if(i == id) continue;
       //获取寻找池子的锁
       acquire(&(bcache.lock[i]));
   
       for(b = bcache.head[i].next, tmp = &(bcache.head[i]); b; b = b->next,tmp = tmp->next)
       {
         if(b->refcnt == 0)
         {
           //找到符合要求的块
           if(b->time < time)
           {
   
             time = b->time;
             last_take = tmp;
             take_buf = b;
             //如果上一个空闲块不在本轮池子中，则释放那个空闲块的锁
             if(lock_num != -1 && lock_num != i && holding(&(bcache.lock[lock_num])))
               release(&(bcache.lock[lock_num]));
             lock_num = i;
           }
         }
       }
       //没有用到本轮池子的块，则释放锁
       if(lock_num != i)
         release(&(bcache.lock[i]));
     }
   
     if (!take_buf)
       panic("bget: no buffers");
   
     //将选中块从其他池子中拿出
     last_take->next = take_buf->next;
     take_buf->next = 0;
     release(&(bcache.lock[lock_num]));
     //将选中块放入本池子中，并写cache
     b = last;
     b->next = take_buf;
     write_cache(take_buf, dev, blockno);
   
   
     release(&(bcache.lock[id]));
     acquiresleep(&(take_buf->lock));
   
     return take_buf;
   }
   ```

   其主要功能是从缓存池中获取指定设备和块号的缓存块。如果该块已经被缓存，那么直接返回该块。如果该块没有被缓存，则从其他缓存池中选择一块最久未使用的空闲块，将其从原缓存池中删除并移动到本缓存池中，并返回该块。

   首先定义了三个指针变量，其中 b 和 last 用于遍历链表，take_buf 用于记录最终选中的缓存块。然后通过 HASH() 函数计算出块号所在的缓存池的编号，并尝试获取该缓存池的锁。

   接下来，从该缓存池中遍历链表，查找是否已经缓存了指定设备和块号的缓存块。如果找到了，则更新该块的时间戳和引用计数，并释放该缓存池的锁，然后等待该缓存块的锁被释放后返回该块。如果没有找到，则记录一个空闲块的指针以备后用。

   如果找到了空闲块，则将其写入指定设备和块号的数据，并返回该块。此时需要先释放该缓存池的锁，然后等待该缓存块的锁被释放后返回该块。

   最后，如果找到了最久未使用的空闲块，则需要将其从原缓存池中删除，并移动到指定设备和块号所在的缓存池中。这里需要获取原缓存池和目标缓存池的锁，并更新链表指针。然后将该块写入指定设备和块号的数据，并释放目标缓存池的锁，最后等待该缓存块的锁被释放后返回该块。

7. 获得了缓存块的时候还上了睡眠锁，对应的 brelse 也要进行修改：

   ```c
   void
   brelse(struct buf *b)
   {
     if(!holdingsleep(&b->lock))
       panic("brelse");
   
     releasesleep(&b->lock);
   
     int h = HASH(b->blockno);
     acquire(&bcache.lock[h]);
     b->refcnt--;
     release(&bcache.lock[h]);
   }
   ```

   函数的主要功能是将一个缓存块标记为不再被使用，并释放该缓存块的锁。具体来说，函数执行以下步骤：

   1. 使用 holdingsleep 函数检查缓存块的锁是否被当前线程持有。如果当前线程没有持有该锁，则调用panic函数触发一个错误并终止程序的执行。
   2. 调用 releasesleep 函数释放缓存块的锁。这样其他线程就可以继续访问这个缓存块了。
   3. 计算缓存块的哈希值，用于确定该缓存块所属的哈希桶。
   4. 调用 acquire 函数获取缓存块所属哈希桶的锁，以防止其他线程同时访问该哈希桶。
   5. 将缓存块的引用计数减一，表示该缓存块被释放了一次。
   6. 调用 release 函数释放缓存块所属哈希桶的锁，以允许其他线程访问该哈希桶。

8. 最后发现还有两个函数与 bcache 的操作相关，一并进行修改：

   ```c
   void bpin(struct buf *b)
   {
     int bucket_id = b->blockno % NBUCKET;
     acquire(&bcache.lock[bucket_id]);
     b->refcnt++;
     release(&bcache.lock[bucket_id]);
   }
   void bunpin(struct buf *b)
   {
     int bucket_id = b->blockno % NBUCKET;
     acquire(&bcache.lock[bucket_id]);
     b->refcnt--;
     release(&bcache.lock[bucket_id]);
   }
   ```

   就是把原本唯一的缓冲区加锁改成了指定桶加锁而已，不是很大的改动。

9. 测试，再次编译并运行 xv6 ，然后运行 bcachetest

   ```cmd
   $ bcachetest
   start test0
   test0 results:
   --- lock kmem/bcache stats
   lock: bcache: #test-and-set 0 #acquire() 4139
   lock: bcache: #test-and-set 0 #acquire() 4150
   lock: bcache: #test-and-set 0 #acquire() 2279
   lock: bcache: #test-and-set 0 #acquire() 4285
   lock: bcache: #test-and-set 0 #acquire() 2271
   lock: bcache: #test-and-set 0 #acquire() 4265
   lock: bcache: #test-and-set 0 #acquire() 4597
   lock: bcache: #test-and-set 207 #acquire() 7506
   lock: bcache: #test-and-set 0 #acquire() 6202
   lock: bcache: #test-and-set 0 #acquire() 6194
   lock: bcache: #test-and-set 0 #acquire() 6194
   lock: bcache: #test-and-set 0 #acquire() 6194
   lock: bcache: #test-and-set 0 #acquire() 6212
   --- top 5 contended locks:
   lock: proc: #test-and-set 5751187 #acquire() 53244820
   lock: proc: #test-and-set 5352803 #acquire() 53224323
   lock: proc: #test-and-set 5133047 #acquire() 53244820
   lock: proc: #test-and-set 5104332 #acquire() 53244820
   lock: proc: #test-and-set 5019627 #acquire() 53244820
   tot= 207
   test0: OK
   start test1
   test1 OK
   ```

   最后 bcachetest 输出 test0 OK 和 test1 OK，且加锁开销远小于改进前的值，符合我们的预期。





## 实验结果

```cmd
yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-lock
== Test running kalloctest == (45.6s)
== Test   kalloctest: test1 ==
  kalloctest: test1: OK
== Test   kalloctest: test2 ==
  kalloctest: test2: OK
== Test kalloctest: sbrkmuch == kalloctest: sbrkmuch: OK (6.0s)
== Test running bcachetest == (7.4s)
== Test   bcachetest: test0 ==
  bcachetest: test0: OK
== Test   bcachetest: test1 ==
  bcachetest: test1: OK
== Test usertests == usertests: OK (87.3s)
== Test time ==
time: OK
Score: 70/70
```

