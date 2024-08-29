# Lab 3 page tables

本实验将探索页表并对其进行修改，以简化将数据从用户空间复制到内核空间的函数

在开始之前，将代码仓库切换到 `pgtbl` 分支



## Experiment 1 : Speed up system calls

#### 1. 实验准备

- 在现实的操作系统中，很多系统调用只是读取内核态的一些数据结构，而并不进行写入操作，这类系统调用的一个典型例子就是 getpid 系统调用。对于这类系统调用，诸如 Linux 等 操作系统不会将内核的数据结构复制到用户空间，而是直接将存放该数据结构的页通过页表的机制映射到用户空间的对应位置，从而免去了复制的开销，提高了系统的性能。

- 本实验中，需要在进程创建时将一个内存页面以只读权限映射到 USYSCALL 位置（参见 memlayout.h）中的定义。该映射的页面开头存储有内核数据结构 struct usyscall ，该数据结构在进程创建时被初始化，并且存储有该进程的 pid 。在这个实验中，我们使用纯用户空间的函数 ugetpid() 来替代需要进行内核态到用户态拷贝的 getpid 系统调用。用户空间的函数 ugetpid() 已经在用户空间中被实现了，我们需要做的是将映射页面的工作完成。

- proc_pagetable() 分配的是一个新的页表，该页表包含多个页表项。而 mappages() 分配的是一个物理页面，并将其映射到一个进程的虚拟地址空间中的一个虚拟地址上。

- PTE_R 和 PTE_U 是 xv6 操作系统中页表项的权限位，它们分别代表“读取”和“用户模式”权限。

  PTE_R 表示该页表项具有读取权限。当该权限位被设置时，进程可以从该页表项所映射的物理页面中读取数据。如果该权限位未被设置，则进程无法从该页表项所映射的物理页面中读取数据。

  PTE_U 表示该页表项可以在用户模式下访问。当该权限位被设置时，进程可以在用户模式下使用该页表项所映射的虚拟地址。如果该权限位未被设置，则该页表项只能在内核模式下使用，进程无法在用户模式下使用该页表项所映射的虚拟地址。

- 注意加锁 : 当在内核态下操作进程相关的数据结构时，由于可能出现并发读写的情况，故而我们需 要先获取该数据结构的锁，然后再进行操作，最后释放该锁；否则可能造成竞争读写，导致数据不一致的情况出现。

#### 2. 实验步骤

1. 按照提示，首先在 kernel/proc.h proc 结构体中添加一项指针来保存这个共享页面的地址

   ```c
   struct proc {
   ...
     struct usyscall *usyscall;  // share page whithin kernel and user
   ...
   }
   ```

2. 由于需要在创建进程时完成页面的映射，故而我们考虑修改 kernel/proc.c 中的 proc_pagetable(struct proc *p) ，即用于为新创建的进程分配页面的函数。在proc_pagetable() 完成分配新的页面后，即可以使用 mappages 函数分配页面。

3. 页面映射时，需要设定其权限为只读，故权限位为 PTE_R | PTE_U ，此后在 allocproc() 中添加初始化该页面，向其中写入数据结构的代码：

   ```c
   // Allocate a usyscall page.
   if((p->usyscall = (struct usyscall *)kalloc()) == 0){
   	freeproc(p);
   	release(&p->lock);
   	return 0;
   }
   p->usyscall->pid = p->pid
   ```

   这段代码是在为一个进程分配一个 usyscall 页面。在 xv6 操作系统中，usyscall 页面是用于实现用户态系统调用的页面，它包含了用户态系统调用的代码和数据。当用户进程发起系统调用时，操作系统会将进程的控制权从用户态切换到内核态，并使用 usyscall 页面中的代码处理系统调用请求。

   具体来说，这段代码的作用如下：

   1. 使用 kalloc() 函数分配一个物理页面，该页面用于存储 usyscall 结构体。
   2. 检查是否分配成功，如果分配失败，则释放进程所占用的资源，并返回 0。
   3. 将 usyscall 结构体的 pid 成员设置为该进程的进程 ID。
   4. 将 usyscall 结构体的指针保存在进程控制块 proc 结构体中的 usyscall 成员中。

   需要注意的是，usyscall 页面是在每个进程创建时分配的，因此该代码段应该在进程创建时被调用。

4. 在 kernel/proc.c 的 proc_pagetable(struct proc *p) 中将这个映射（PTE）写入 pagetable 中。权限是用户态可读。

   ```c
     //map a user read only page at USYSCALL, for optimization for the getpid()
     if(mappages(pagetable, USYSCALL, PGSIZE,
                 (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
       uvmfree(pagetable, 0);
       return 0;
     }
   ```

5. 在完成页面的初始化后，进程应当得以正常运行，但需要记得在进程结束后释放分配的 页面，该动作在 freeproc() 中进行，添加的代码如下：

   ```c
   if(p->usyscall)
       kfree((void*)p->usyscall);
   p->usyscall = 0;
   ```

6. 当考虑好进程的页面分配，页面初始化和页面回收，页面映射后，映射页面的工作就全部完成了。编译并启动 xv6 后，会报错：`panic: freewalk: leaf`

   这是 xv6 操作系统中的一种错误消息，表示在释放页表时出现了错误。就是不仅要释放内存，还要取消这个映射关系。如果只释放了页表不取消这些映射关系，那么其他进程或者内核代码可能会使用这些虚拟地址来访问已经释放的物理页面，从而导致内存安全问题之类的。

   ```c
   void
   proc_freepagetable(pagetable_t pagetable, uint64 sz)
   {
     uvmunmap(pagetable, TRAMPOLINE, 1, 0);
     uvmunmap(pagetable, TRAPFRAME, 1, 0);
     uvmunmap(pagetable, USYSCALL, 1, 0); // add
     uvmfree(pagetable, sz);
   }
   ```

7. 测试

   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-pgtbl ugetpid
   make: 'kernel/kernel' is up to date.
   == Test pgtbltest == (0.7s)
   == Test   pgtbltest: ugetpid ==
     pgtbltest: ugetpid: OK
   ```





## Experiment 2 : Print a page table

#### 1. 实验准备

-  为了帮助了解RISC-V页表和将来的调试，我们的第一个任务是编写一个打印页表内容的函数。
-  定义一个名为`vmprint()`的函数。它应当接收一个`pagetable_t`作为参数，并以下面描述的格式打印该页表。在`exec.c`中的`return argc`之前插入`if(p->pid==1) vmprint(p->pagetable)`，以打印第一个进程的页表。
- 研究`kernel/vm.c`中的`freewalk`函数，在`freewalk`函数中，首先会遍历整个`pagetable`，当遍历到一个有效的页表项且该页表项不是最后一层时，会`freewalk`函数会递归调用。`freewalk`函数通过`(pte & PTE_V)`判断页表是否有效，`(pte & (PTE_R|PTE_W|PTE_X)) == 0`判断是否在不在最后一层。

#### 2. 实验步骤

1. 在`kernel/defs.h`中定义函数原型

   ```c
   // exec.c
   int             exec(char*, char**);
   void            vmprint(pagetable_t);
   ```

2. 在`exec.c`中的`return argc`之前插入`if(p->pid==1) vmprint(p->pagetable)`

   ```c
   // kernel/exec.c
   // lab 3.1
   if(p->pid==1)
       vmprint(p->pagetable);
   ```

3. 参考`kernel/vm.c`中的`freewalk`函数，我们可以在`kernel/vm.c`中写出`vmprint`函数的递归实现

   ```c
   void
   vmprint_rec(pagetable_t pagetable, int level){
     // there are 2^9 = 512 PTEs in a page table.
     for(int i = 0; i < 512; i++){
       pte_t pte = pagetable[i];
       // PTE_V is a flag for whether the page table is valid
       if(pte & PTE_V){
         for (int j = 0; j < level; j++) {
           if (j) 
             printf(" ");
           printf("..");
         }
         uint64 child = PTE2PA(pte);
         printf("%d: pte %p pa %p\n", i, pte, child);
         if((pte & (PTE_R|PTE_W|PTE_X)) == 0) {
           // this PTE points to a lower-level page table.
           vmprint_rec((pagetable_t)child, level + 1);
         }
       }
     }
   }
   
   void
   vmprint(pagetable_t pagetable) {
     printf("page table %p\n", pagetable);
     vmprint_rec(pagetable, 1);
   }
   ```

    上述代码中，`vmprint_rec`是递归函数，`level`传入层级来根据层级数量输出".."，`vmprint`是真正的输出函数，通过调用递归函数`vmprint_rec`实现。

4. 测试

   先`make clean`去掉旧的依赖关系，然后`make qemu`，发现有如下输出结果

   注意，一定要`make clean`不然会报错：``make: *** No rule to make target  kernel/sysinfo.h , needed by  kernel/sysproc.o .  Stop.``
   
   ```cmd
   hart 2 starting
   hart 1 starting
   page table 0x0000000087f6e000
    ..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
    .. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
    .. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
    .. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
    .. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
    ..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
    .. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
    .. .. ..509: pte 0x0000000021fddc13 pa 0x0000000087f77000
    .. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
    .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
   init: starting sh
   ```
   
   说明个人测试成功，下面进行单元测试，输出如下结果
   
   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-pgtbl pte printout
   make: 'kernel/kernel' is up to date.
   == Test pte printout == pte printout: OK (1.1s)
   ```
   
   



### Experiment 3 : Detecting which pages have been accessed

#### 1. 实验准备

- 现代很多编程语言都具备了内存垃圾回收的功能（GC ，garbage collect ），而垃圾回收器功能的实现则需要得到一个页面自上次检查后是否被访问的信息。实现这种功能可以是纯软 件的，但其效率低下；而利用页表的硬件机制（访问位）和操作系统相结合，在 xv6 中添加一个系统调用 pgaccess() 用以读取页表的访问位并传递给用户态程序，从而告知他们一些内存页面自上次检查以来的访问情况，是一个较好的方法。
- 我们需要实现的 pgaccess() 系统调用接受 3 个参数：第一个参数为需要检查页面的初始 地址，第二个参数是从初始地址向后检查多少个页面，第三个参数则是一个指针，指向用于存储结果的位向量的首地址。

#### 2. 实验步骤

1. 首先获取用户参数，包括待访问的虚拟地址 srcva、待访问的地址空间长度 len、用于保存访问标志的缓冲区起始地址 st。
2. 对于每一个地址空间页（即每个虚拟地址），调用 walk() 函数查找该地址对应的页表项 pte。如果该页表项不存在或无效，则返回错误码 -1。
3. 检查页表项的权限：如果该页表项不具有用户权限（即 PTE_U 标志位未设置），则返回错误码 -1。
4. 检查页表项的访问标志位：如果该页表项的访问标志位 PTE_A 已经被设置，则将其清除，并将这个页面的编号记录到缓冲区 buf 中。（其中PTE_A需要在 kernel/riscv.h 中宏定义一下#define PTE_A (1L << 6) // 1 -> access bit 代表在6号位置的访问位）
5. 在访问完所有的虚拟地址后，调用 copyout() 函数将缓冲区 buf 中的访问标志位状态复制到用户空间。
6. 最后释放进程锁，返回错误码 0 表示访问成功。

```cmd
extern pte_t *walk(pagetable_t, uint64, int);
int
sys_pgaccess(void)
{
  // lab pgtbl: your code here.
  uint64 srcva, st;
  int len;
  uint64 buf = 0;
  struct proc *p = myproc();

  acquire(&p->lock);

  argaddr(0, &srcva);
  argint(1, &len);
  argaddr(2, &st);
  if ((len > 64) || (len < 1))
    return -1;
  pte_t *pte;
  for (int i = 0; i < len; i++)
  {
    pte = walk(p->pagetable, srcva + i * PGSIZE, 0);
    if(pte == 0){
      return -1;
    }
    if((*pte & PTE_V) == 0){
      return -1;
    }
    if((*pte & PTE_U) == 0){
      return -1;
    }
    if(*pte & PTE_A){
      *pte = *pte & ~PTE_A;
      buf |= (1 << i);
    }
  }
  release(&p->lock);
  copyout(p->pagetable, st, (char *)&buf, ((len -1) / 8) + 1);
  return 0;
}
```





## 实验结果

```cmd
yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-pgtbl
make: 'kernel/kernel' is up to date.
== Test pgtbltest == (0.8s)
== Test   pgtbltest: ugetpid ==
  pgtbltest: ugetpid: OK
== Test   pgtbltest: pgaccess ==
  pgtbltest: pgaccess: OK
== Test pte printout == pte printout: OK (0.9s)
== Test answers-pgtbl.txt == answers-pgtbl.txt: OK
== Test usertests == (73.0s)
== Test   usertests: all tests ==
  usertests: all tests: OK
== Test time ==
time: OK
Score: 46/46
```

