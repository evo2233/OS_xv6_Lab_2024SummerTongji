# Lab 5 Copy-on-Write Fork for xv6

写时复制（COW ）机制是操作系统中一种常见的惰性机制，在很多场景下可以提供较好 的性能和比物理资源多很多的虚拟资源量，因而机制在内存管理中使用较多。实现 COW 机制 需要我们将前几个实验的涉及的概念（进程、分页和中断）融会贯通，从这个实验开始，我们 的工作将遍及 xv6 内核的各个部分。

开始之前切换到 cow 分支



## Experiment 1 : Implement copy-on write

#### 1. 实验准备

我们早在第三章 Lab Utilities 中就已经接触了 xv6 中用于生成子进程的系统调用 fork ，该系统调用由于其实用的设计被后世的类 Unix 系统一直沿用，可谓 Unix v6 留下的宝贵遗产。在原始的 xv6 的实现中，fork 会将父进程的所有页面完整地复制到一份，映射到子进程中。然而很多时候，子进程只会读取这些页面的内容，而不会写入这些页面。为了节约内存的实际用量，我们可以在子进程（或父进程）真正需要写入页面时才分配新的页面并进行数据的复制，这种机制被称为 COW fork 。

- 下面是一些建议：
  1. 修改 uvmcopy() ，将父进程的物理页映射到子进程，以避免分配新的页。同时清楚子进程和父进程 PTE 的 PTE_W 位，使它们标记为不可写。
  2. 修改 usertrap() ，以此来处理页面错误 (page fault)。当在一个本来可写的 COW 页发生“写页面错误” (write page-falut) 的时候，使用 kalloc() 分配一个新的页，将旧页中的数据拷贝到新页中，并将新页的 PTE 的 PTE_W 位置为1. **值得注意的是，对于哪些本来就使只读的（例如代码段），不论在旧页还是新页中，应该依旧保持它的只读性，那些试图对这样一个只读页进行写入的进程应该被杀死。**
  3. 确保没有一个物理页被 PTE 引用之后，再释放它们，不能提前释放。一个好的做法是，维持一个“引用计数数组” (reference count) ，使用索引代表对应的物理页，值代表被引用的个数。当调用 kalloc() 分配一个物理页的时候，将其 reference count 设置为 1 。当 fork 导致子进程共享同一个页的时候，递增其页的引用计数；当任何一个进程从它们的页表中舍弃这个页的时候，递减其页的引用计数。kfree() 仅当一个页的引用计数为零时，才将这个页放到空闲页列表中。将这些引用计数放到一个 int 类型的数组中是可以的，你必须思考，如何去索引这个数组，以及如何选择这个数组的大小。例如，你可以用一个页的物理地址/4096 来索引数组。
  4. 修改 copyout() ，使用类似的思路去处理 COW 页。
- 一些注意的点：
  1. 记录一个 PTE 是否是 COW 映射是有帮助的。你可以使用 RISC-V PTE 的 RSW (reserved for software) 位来指示。
  2. usertests -q 测试一些 cowtest 没有测试的东西，所以不要忘记两个测试都要通过。
  3. 一些对于页表标志 (page table flags) 有用的宏和定义可以在 kernel/riscv.h 中找到。
  4. 如果一个 COW 页错误发生时，没有剩余可用内存，那么该进程应该被杀死。

#### 2. 实验步骤

1. 首先需要处理引用计数的问题。在 kernel/kalloc.c 中定义一个全局变量和锁。

   ```c
   // the reference count of physical memory page
   int useReference[PHYSTOP/PGSIZE];
   struct spinlock ref_count_lock;
   ```

2. 然后在函数 kalloc() 中，初始化新分配的物理页的引用计数为 1。

   ```c
   void *
   kalloc(void)
   {
     ...
     if(r) {
       kmem.freelist = r->next;
       acquire(&ref_count_lock);
       // initialization the ref count to 1
       useReference[(uint64)r / PGSIZE] = 1;
       release(&ref_count_lock);
     }
     release(&kmem.lock);
     ...
   }
   ```

   在该代码段中，if (r) 是一个条件语句，用于判断 kmem.freelist 是否为空。kmem.freelist 是一个指向空闲物理内存页面链表的指针。如果 kmem.freelist 不为空，则说明有空闲的物理内存页面可以被分配给进程使用。

3. 接着修改 kfree() ，仅当引用计数小于等于 0 的时候，才回收对应的页。

   ```c
   void
   kfree(void *pa)
   {
     ...
     if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
       panic("kfree");
   	
   	int temp;
     acquire(&ref_count_lock);
     // decrease the reference count, if use reference is not zero, then return
     useReference[(uint64)pa/PGSIZE] -= 1;
     temp = useReference[(uint64)pa/PGSIZE];
     release(&ref_count_lock);
     if (temp > 0)
       return;
   
     // Fill with junk to catch dangling refs.
    ...
   }
   ```

   之所以不使用 if (temp != 0) ，因为在系统运行开始的时候，需要对空闲页列表 (kmem.freelist) 进行初始化，此时的引用计数就为 -1 ，如果条件为 temp != 0 那么这些空闲页就不能够回收，也就不能够进 kmem.freelist 列表了。

4. fork 会首先调用 uvmcopy() 给子进程分配内存空间。但是如果要实现 COW 机制，就需要在 fork 时不分配内存空间，而是让子进程和父进程同时共享父进程的内存页，并将其设置为只读，使用 PTE_RSW 位标记 COW 页。这样子进程没有使用到某些页的时候，系统就不会真正的分配物理内存。注意，此时需要将对应的引用计数加一。

5. 首先在 kernel/riscv.h 中增加一些定义：

   ```c
   #define PTE_RSW (1L << 8) // RSW
   ```

6. 然后修改 uvmcopy 函数。此函数在 vm.c 文件中，由于需要使用 useReference 引用计数，所以需要提前在文件开头声明。

   ```c
   #include "spinlock.h"
   #include "proc.h"
   
   // Just declare the variables from kernel/kalloc.c
   extern int useReference[PHYSTOP/PGSIZE];
   extern struct spinlock ref_count_lock;
   
   int
   uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
   {
     ...
     // char *mem;
     for(i = 0; i < sz; i += PGSIZE){
       if((pte = walk(old, i, 0)) == 0)
         panic("uvmcopy: pte should exist");
       if((*pte & PTE_V) == 0)
         panic("uvmcopy: page not present");
       // PAY ATTENTION!!!
       // 只有父进程内存页是可写的，才会将子进程和父进程都设置为COW和只读的；否则，都是只读的，但是不标记为COW，因为本来就是只读的，不会进行写入
       // 如果不这样做，父进程内存只读的时候，标记为COW，那么经过缺页中断，程序就可以写入数据，于原本的不符合
       if (*pte & PTE_W) {
         // set PTE_W to 0
         *pte &= ~PTE_W;
         // set PTE_RSW to 1
         // set COW page
         *pte |= PTE_RSW;
       }
       pa = PTE2PA(*pte);
   
       // increment the ref count
       acquire(&ref_count_lock);
       useReference[pa/PGSIZE] += 1;
       release(&ref_count_lock);
   
       flags = PTE_FLAGS(*pte);
       // if((mem = kalloc()) == 0)
       //   goto err;
       // memmove(mem, (char*)pa, PGSIZE);
       if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
         // kfree(mem);
         goto err;
       }
     }
     ...
   }
   ```

   代码的具体实现过程如下：

   1. 该段代码通过循环遍历源进程中需要复制的所有内存页面，每次复制一个页面的大小（PGSIZE）。
   2. 对于每个页面，首先调用 walk() 函数在源进程的页表中查找对应的页表项，如果该页表项不存在，则出错并调用 panic() 函数结束程序。如果该页面不在内存中，则也会出错并调用 panic() 函数。
   3. 如果该页面是可写的，则将 PTE_W 标志位设置为0，将 PTE_RSW 标志位设置为1，从而将该页面标记为COW页面。如果该页面是只读的，则不需要标记为COW页面，因为只读页面不会被修改。
   4. 接下来，程序会获取该页面对应的物理地址，然后增加该物理页面的引用计数，以便跟踪该页面被使用的情况。
   5. 最后，程序调用 mappages() 函数将该物理页面映射到目标进程的虚拟地址空间中。如果映射失败，则出错并跳转到错误处理代码块。

   一旦子进程真正需要某些页，由于页被设置为只读的，此时会触发 Store/AMO page fault，scause 寄存器的值为 15。

7. 在 usertrap() 函数中捕获到 Store/AMO page fault 错误之后开始处理。首先应该知道哪个虚拟地址的操作导致了页错误。RISC-V 中的 stval (Supervisor Trap Value (stval) Register ) 寄存器中的值是导致发生异常的虚拟地址。vx6 中的函数 r_stval() 可以获取该寄存器的值。由此修改 usertrap 函数：

   ```c
   void
   usertrap(void)
   {
     ...
     syscall();
     }
     else if (r_scause() == 15) {
       // Store/AMO page fault(write page fault)
       // see Volume II: RISC-V Privileged Architectures V20211203 Page 71
   
       // the faulting virtual address
       // see Volume II: RISC-V Privileged Architectures V20211203 Page 70
       // the download url is https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf
       uint64 va = r_stval();
       if (va >= p->sz)
         p->killed = 1;
       int ret = cowhandler(p->pagetable, va);
       if (ret != 0)
         p->killed = 1;
     } else if((which_dev = devintr()) != 0){
       // ok
     } else {
     ...
    }
   ```

   代码的具体实现过程如下：

   1. 首先，程序通过 r_scause() 函数获取最近一次发生的异常或中断的原因代码，如果代码是15，表示最近一次发生了一个 Store/AMO page fault 异常。
   2. 接下来，程序调用 r_stval() 函数获取最近一次 Store/AMO page fault 异常的触发地址 va。该地址是虚拟地址，代表程序试图写入的内存页面的地址。
   3. 如果该地址大于等于进程的大小 p->sz，则说明程序试图写入的地址超出了进程的地址空间，因此将进程的 killed 标志设置为1，表示该进程已经被终止。
   4. 否则，程序调用 cowhandler() 函数对该地址所对应的内存页面进行 COW（Copy-On-Write，写时复制）操作。
   5. 如果 COW 操作成功，则返回值为0，表示程序可以继续执行。否则，将进程的 killed 标志设置为1，表示该进程已经被终止。

8. 定义其中用到的 cowhandler 函数：

   ```c
   int
   cowhandler(pagetable_t pagetable, uint64 va)
   {
       char *mem;
       if (va >= MAXVA)
         return -1;
       pte_t *pte = walk(pagetable, va, 0);
       if (pte == 0)
         return -1;
       // check the PTE
       if ((*pte & PTE_RSW) == 0 || (*pte & PTE_U) == 0 || (*pte & PTE_V) == 0) {
         return -1;
       }
       if ((mem = kalloc()) == 0) {
         return -1;
       }
       // old physical address
       uint64 pa = PTE2PA(*pte);
       // copy old data to new mem
       memmove((char*)mem, (char*)pa, PGSIZE);
       // PAY ATTENTION
       // decrease the reference count of old memory page, because a new page has been allocated
       kfree((void*)pa);
       uint flags = PTE_FLAGS(*pte);
       // set PTE_W to 1, change the address pointed to by PTE to new memory page(mem)
       *pte = (PA2PTE(mem) | flags | PTE_W);
       // set PTE_RSW to 0
       *pte &= ~PTE_RSW;
       return 0;
   }
   ```

   具体分析如下：

   1. 首先，定义一个名为 cowhandler 的函数，该函数接受两个参数：pagetable_t 类型的页表和 uint64 类型的虚拟地址 va。函数返回一个整数类型的值，表示操作是否成功。
   2. 定义一个指针类型为 char 的变量 mem，用于存储分配到的新的物理内存页面的地址。
   3. 如果虚拟地址 va 大于等于 MAXVA（最大虚拟地址），则返回-1，表示操作失败。
   4. 调用 walk() 函数，查找页表中虚拟地址 va 所指向的页表项 PTE，并将其保存在 pte 指针变量中。第二个参数为0，表示不创建新的页表项。
   5. 如果找不到页表项 PTE，则返回-1，表示操作失败。
   6. 检查 PTE 的标志位，如果 PTE_RSW（Reserved for software use，保留给软件使用）位等于0，或 PTE_U（User，用户）位等于0，或 PTE_V（Valid，有效）位等于0，则返回-1，表示操作失败。
   7. 调用 kalloc() 函数，为新的物理内存页面分配内存。如果分配失败，则返回-1，表示操作失败。
   8. 使用 PTE2PA() 宏将页表项 PTE 中的物理地址转换为 uint64 类型的 pa 变量中。
   9. 调用 memmove() 函数，将原始物理内存页面的内容复制到新的物理内存页面中，从而实现 COW 操作。
   10. 调用 kfree() 函数，减少原始物理内存页面的引用计数，因为现在存在一个新的物理内存页面。
   11. 使用 PTE_FLAGS() 宏获取页表项 PTE 的标志位，并将其保存在 flags 变量中。
   12. 将新的物理内存页面的地址 mem 转换为页表项 PTE，使用 PA2PTE() 宏将其转换为 PTE 类型的值。将 PTE_W（Writable，可写）位设置为1，表示该页面是可写的。最后将新的 PTE 值赋值给原始的 PTE 指针。
   13. 将 PTE_RSW 位设置为0，表示该位不再保留给软件使用。
   14. 返回0，表示操作成功。

   cowhandler 要进行严格的检查。只有被标记为 COW，存在，且是属于用户级别的，才可以被分配内存。如果本来的页就是只读的，那么在此时尝试对其进行写入，就会返回 -1，最终被杀死。

9. 接着修改 kernel.vm.c 中的 copyout() 函数（我们之前见到过，是用于将内核地址空间中的数据复制到用户地址空间中的函数）。需要修改 copyout() 使得其能够适应 COW 机制：因为 copyout() 是在内核态修改用户态的页面内存，故而不会引发从用户态来的违例访问(权限高)，因此需要我们做特殊处理。这里的思路和 uvmcopy() 中类似:

   ```c
   int checkcowpage(uint64 va, pte_t *pte, struct proc* p) {
     return (va < p->sz) // va should blow the size of process memory (bytes)
       && (*pte & PTE_V) 
       && (*pte & PTE_RSW); // pte is COW page
   }
   
   int
   copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
   {
       ...
       pa0 = walkaddr(pagetable, va0);
       if(pa0 == 0)
         return -1;
   
       struct proc *p = myproc();
       pte_t *pte = walk(pagetable, va0, 0);
       if (*pte == 0)
         p->killed = 1;
       // check
       if (checkcowpage(va0, pte, p)) 
       {
         char *mem;
         if ((mem = kalloc()) == 0) {
           // kill the process
           p->killed = 1;
         }else {
           memmove(mem, (char*)pa0, PGSIZE);
           // PAY ATTENTION!!!
           // This statement must be above the next statement
           uint flags = PTE_FLAGS(*pte);
           // decrease the reference count of old memory that va0 point
           // and set pte to 0
           uvmunmap(pagetable, va0, 1, 1);
           // change the physical memory address and set PTE_W to 1
           *pte = (PA2PTE(mem) | flags | PTE_W);
           // set PTE_RSW to 0
           *pte &= ~PTE_RSW;
           // update pa0 to new physical memory address
           pa0 = (uint64)mem;
         }
       }
   
       n = PGSIZE - (dstva - va0);
       if(n > len)
         n = len;
       ...
   }
   ```

   使用 checkcowpage 检测 PTE 的标志。uvmunmap 函数将 PTE 置零，并 kfree PTE 引用的物理页，其实这里的 kfree 就是将 PTE 引用的物理页的引用计数减一。因为 copyout 分配的一个新的物理页，所以不需要共享之前的旧页了。

   由于 uvmunmap 将 PTE 置零，所以 uint flags = PTE_FLAGS(*pte); 必须在其前面。





## 实验结果

```cmd
yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-cow
make: 'kernel/kernel' is up to date.
== Test running cowtest == (2.9s)
== Test   simple ==
  simple: OK
== Test   three ==
  three: OK
== Test   file ==
  file: OK
== Test usertests == (72.1s)
== Test   usertests: copyin ==
  usertests: copyin: OK
== Test   usertests: copyout ==
  usertests: copyout: OK
== Test   usertests: all tests ==
  usertests: all tests: OK
== Test time ==
time: OK
Score: 110/110
```

