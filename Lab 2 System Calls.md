# Lab 2 ：System Calls

在`Lab1`中，我们使用系统调用编写了一些实用的用户态程序，在这个实验中，我们将为`xv6`添加一些新的系统调用，以便更深入理解`xv6`系统

在开始之前，将代码仓库切换到 `syscall` 分支



## Experiment 1 : System call tracing

#### 1. 实验准备

- `trace`介绍：实现一个可以追踪任意系统调用的系统调用，当设置了追踪掩码mask之后，对应编号的系统调用在触发时都会被追踪。并打印出如下的信息：``进程ID号：syscall xxx -> 返回值``

- `xv6`已经实现了一个文件`trace.c`，这个文件和我们在上一个实验中实现的用户态程序是一样的，`trace.c`会调用`trace`系统调用来跟踪后面具体执行的命令行，所以这里也需要将`_trace`加入`UPROGS`字段，否则找不到这个应用程序。

- 用到的一些函数

  - `argint` : 用于从系统调用的参数中提取整型值，用于在系统调用处理函数中获取传递给系统调用的整数参数

    - 输入

      ```c
      int num;
      argint(0, &num); // 获取第一个参数，并将其存储到 num 中
      ```

    - 输出

      返回从系统调用参数中提取出的整型值

  - `myproc` : 用于获取当前正在执行的进程的进程结构体指针，使得进程管理中的其他函数可以访问当前进程的相关信息

    - 输出：返回一个指向当前进程的进程控制块（`proc` 结构体）的指针
  
- 完成实验后：

  ```cmd
  $ trace 32 grep hello README
  3: syscall read -> 1023
  3: syscall read -> 968
  3: syscall read -> 235
  3: syscall read -> 0
  $ trace 2147483647 grep hello README
  4: syscall trace -> 0
  4: syscall exec -> 3
  4: syscall open -> 3
  4: syscall read -> 1023
  4: syscall read -> 968
  4: syscall read -> 235
  4: syscall read -> 0
  4: syscall close -> 0
  $ grep hello README
  $ trace 2 usertests forkforkfork
  usertests starting
  6: syscall fork -> 7
  test forkforkfork: 6: syscall fork -> 8
  8: syscall fork -> 9
  9: syscall fork -> 10
  9: syscall fork -> 11
  10: syscall fork -> 12
  9: syscall fork -> 13
  11: syscall fork -> 14
  10: syscall fork -> 15
  12: syscall fork -> 16
  ```

  在上面的第一个例子中，trace 调用 grep 仅跟踪 read 系统调用。32 是 1<<SYS_read。

  在第二个例子中，trace 在跟踪所有系统调用的同时运行 grep；2147583647 的所有 31 个低位均已设置。

  在第三个例子中，程序未被跟踪，因此没有打印任何跟踪输出。

  在第四个例子中，正在跟踪 usertests 中 forkforkfork 测试的所有后代的 fork 系统调用。

#### 2. 实验步骤

1. 编写 trace 的系统调用函数

   - 在`kernel/sysproc.c`中添加一个`sys_trace()`函数，通过在`proc`结构（参见`kernel/proc.h`）中的一个新变量`tracemask`来存储参数来实现新的系统调用
   - 放哪个内核的文件夹主要起归纳作用，这里因为我们的系统调用会对进程进行操作，所以放在 sysproc.c 较为合适

   ```c
   uint64 sys_trace(void) {
       int mask;
       // 通过读取进程的 trapframe，获得 mask 参数
       if (argint(0, &mask) < 0) {
           return -1;
       }
       // 设置调用进程的 tracemask
       myproc()->tracemask = mask;
       return 0;
   }
   ```

2. 在 `kernel/syscall.h` 中添加 trace 的系统调用号

   ```c
   #define SYS_trace 22	// lab 2.1
   ```

3. 用 extern 全局声明新的内核调用函数，并且在 syscall 映射表中，加入从前面定义的编号到系统调用函数指针的映射

   ```c
   // kernel/syscall.c
   extern uint64 sys_trace(void);	// lab 2.1
   ...
   [SYS_trace]   sys_trace,  // lab 2.1
   ```

   这里 [SYS_trace] sys_trace 是 C 语言数组的一个语法，表示以方括号内的值作为元素下标。比如 int arr[] = {[3] 2333, [6] 6666} 代表 arr 的下标 3 的元素为 2333，下标 6 的元素为 6666，其他元素填充 0 的数组。

4. 用户态到内核态的跳板函数

   - 在 `user/usys.pl` 脚本中添加 trace 对应的 entry

     这个脚本在运行后会生成 usys.S 汇编文件，里面定义了每个 system call 的用户态到内核态的跳板函数

   ```perl
   # user/usys.pl
   entry("trace");	# lab 2.1
   ```

   - 在用户态的头文件加入定义，使得用户态程序可以找到这个跳板入口函数

   ```c
   // user/user.h
   // system calls
   int trace(int);	// lab 2.1
   ```

5. 经过上述过程，我们完成了系统调用的全流程：

   ```bash
   user/user.h:		用户态程序调用跳板函数 trace()
   user/usys.S:		跳板函数 trace() 使用 CPU 提供的 ecall 指令，调用到内核态
   kernel/syscall.c	到达内核态统一系统调用处理函数 syscall()，所有系统调用都会跳到这里来处理。
   kernel/syscall.c	syscall() 根据跳板传进来的系统调用编号，查询 syscalls[] 表，找到对应的内核函数并调用。
   kernel/sysproc.c	到达 sys_trace() 函数，执行具体内核操作
   ```

   - 这么繁琐的调用流程的主要目的是实现用户态和内核态的良好隔离

   - 并且由于内核与用户进程的页表不同，寄存器也不互通，所以参数无法直接通过 C 语言参数的形式传过来，而是需要使用 argaddr、argint、argstr 等系列函数，从进程的 trapframe 中读取用户进程寄存器中的参数。

   - 同时由于页表不同，指针也不能直接互通访问（也就是内核不能直接对用户态传进来的指针进行解引用），而是需要使用 copyin、copyout 方法结合进程的页表，才能顺利找到用户态指针（逻辑地址）对应的物理内存地址。

6. 修改 `struct proc` 的定义，添加 tracemask field，用 mask 的方式记录要 trace 的 system call

   ```c
   // kernel/proc.h
   uint64 tracemask;        // Mask for syscall tracing (Lab 2.1)
   ```

7. 创建新进程的时候，为新添加的 syscall_trace 附上默认值 0

   ```c
   // kernel/proc.c/allocproc()
   p->tracemask = 0; // Lab 2.1
   ```

8. 修改 fork 函数，复制父进程的跟踪掩码到子进程，使得子进程可以继承父进程的 syscall_trace mask

   ```c
   // kernel/proc.c
   np->tracemask = p->tracemask; // Lab 2.1
   ```

9. 根据上方提到的系统调用的全流程，可以知道，所有的系统调用到达内核态后，都会进入到 syscall() 这个函数进行处理，所以要跟踪所有的内核函数，只需要在 syscall() 函数里埋点就行了

   ```c
   // kernel/syscall.c
   /*-Lab 2.1-*/
   if ((1 << num) & p->tracemask) {    // 判断掩码是否匹配
     printf("%d: syscall %s -> %d\n", p->pid, syscalls_name[num], p->trapframe->a0);
   }
   /*-Lab 2.1-*/
   ```

10. 上面打出日志的过程还需要知道系统调用的名称字符串，在这里定义一个字符串数组进行映射

    ```c
    // kernel/syscall.c
    // lab 2.1
    static char* syscalls_name[] = {
            "",
            "fork",
            "exit",
            "wait",
            "pipe",
            "read",
            "kill",
            "exec",
            "fstat",
            "chdir",
            "dup",
            "getpid",
            "sbrk",
            "sleep",
            "uptime",
            "open",
            "write",
            "mknod",
            "unlink",
            "link",
            "mkdir",
            "close",
            "trace",    // lab 2.1
    };
    ```

11. 在 Makefile 中添加

    ```c
    $U/_trace\
    ```

    测试：

    ```cmd
    yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-syscall trace
    make: 'kernel/kernel' is up to date.
    == Test trace 32 grep == trace 32 grep: OK (0.9s)
    == Test trace all grep == trace all grep: OK (0.7s)
    == Test trace nothing == trace nothing: OK (1.0s)
    == Test trace children == trace children: OK (8.4s)
    ```
    





## Experiment 2 : Sysinfo

#### 1. 实验准备

- `sysinfo`介绍：它收集有关正在运行的系统的信息。系统调用采用一个参数：一个指向`struct sysinfo`的指针（参见`kernel/sysinfo.h`）。内核应该填写这个结构的字段：`freemem`字段应该设置为空闲内存的字节数，`nproc`字段应该设置为`state`字段不为`UNUSED`的进程数。

- **不要忘记实现一个系统调用的必经4步骤**

- 完成实验后，利用提供的测试程序 `sysinfotest`本地测试

  ```cmd
  $ sysinfotest
  sysinfotest: start
  sysinfotest: OK
  ```

#### 2. 实验步骤

1. 与上一个实验类似，在`user/user.h`中添加`sysinfo()`系统调用声明和`struct sysinfo`结构体声明

   ```c
   struct sysinfo;     // lab 2.2
   ...
   // system calls
   int sysinfo(struct sysinfo*); // lab 2.2
   ```

2. 在`user/usys.pl`中添加`sysinfo`系统调用的入口

   ```perl
   entry("sysinfo"); # lab 2.2
   ```

3. 在`kernel/syscall.h`中添加系统调用号的宏定义

   ```c
   #define SYS_sysinfo 23 // lab 2.2
   ```

4. 在内核中声明系统调用：先在`kernel/syscall.c`中，新增`sys_sysinfo`函数原型和定义，由于在上一个实验中新增了`syscalls_name`数组以记录各个系统调用的名字，所以需要在`syscalls_name`数组中新增"sysinfo"这个名字

   ```c
   extern uint64 sys_sysinfo(void); // lab 2.2
   ...
   [SYS_sysinfo]   sys_sysinfo, // lab 2.2
   ...
   "sysinfo"   // lab 2.2
   ```

5. 实现`sys_sysinfo`函数：

   根据实验提示，`sys_sysinfo`函数需要将一个`struct sysinfo`复制回用户空间，具体实现可以参考`sys_fstat()`和`filestat()`是如何使用`copyout()`的。`sys_sysinfo`函数实现如下，该函数需要添加在`kernel/sysproc.c`中。

   ```c
   //kernel/sysproc.c
   #include "sysinfo.h"	// lab 2.2
   // Lab 2.2
   uint64
   sys_sysinfo(void)
   {
     uint64 addr;
     struct sysinfo info;
     struct proc *p = myproc();
     
     if (argaddr(0, &addr) < 0)
   	  return -1;
     info.freemem = free_mem();
     info.nproc = proc_number();
     //copy
     if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
       return -1;
     
     return 0;
   }
   ```

6. 在 `kernel/kalloc.c` 中添加一个`free_mem`函数，用于收集可用内存的字节数

   ```c
   // Lab 2.2
   uint64
   free_mem(void)
   {
     struct run *p;
     // num: free page number
     uint64 num = 0;
     // spinlock
     acquire(&kmem.lock);
     p = kmem.freelist;  //work pointer
     // travel node list
     while (p)
     {
       num++;
       p = p->next;
     }
     // spinlock
     release(&kmem.lock);
     // memory size = num*PGSIZE
     return num * PGSIZE;
   }
   ```

7. 在 `kernel/proc.c` 中添加一个`proc_number`函数，用于收集进程数：

   ```c
   // Lab 2.2
   uint64
   proc_number(void)
   {
     struct proc *p;
     uint64 num = 0;
     // visit all process
     for (p = proc; p < &proc[NPROC]; p++)
     {
       // lock
       acquire(&p->lock);
       if (p->state != UNUSED)
       {
         num++;
       }
       // lock
       release(&p->lock);
     }
     return num;
   }
   ```

8. 在 `kernel/defs.h` 中添加上述两个函数原型。

   ```c
   // kalloc.c
   uint64          free_mem(void);   // lab2.2
   // proc.c
   uint64          proc_number(void);     // lab2.2
   ```

9. 在`user`目录下新增一个`sysinfo.c`的用户程序

   ```c
   #include "kernel/types.h"
   #include "kernel/param.h"
   #include "kernel/sysinfo.h"
   #include "user/user.h"
   
   int
   main(int argc, char *argv[])
   {
       if (argc != 1)
       {
           fprintf(2, "Usage: %s need not param\n", argv[0]);
           exit(1);
       }
   
       struct sysinfo info;
       sysinfo(&info);
       
       printf("free space: %d\nused process: %d\n", info.freemem, info.nproc);
       exit(0);
   }
   ```

10. 在 Makefile 的 `UPROGS` 中添加 `$U/_sysinfotest\`

11. 测试：

    ```cmd
    yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-syscall sysinfo
    ...
    == Test sysinfotest == sysinfotest: OK (2.7s)
    ```





## 实验结果

```cmd
yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-syscall
make: 'kernel/kernel' is up to date.
== Test trace 32 grep == trace 32 grep: OK (1.3s)
== Test trace all grep == trace all grep: OK (0.4s)
== Test trace nothing == trace nothing: OK (0.7s)
== Test trace children == trace children: OK (8.3s)
== Test sysinfotest == sysinfotest: OK (1.0s)
== Test time ==
time: OK
Score: 35/35
```

最后上传这个分支到Github仓库即可完成实验。