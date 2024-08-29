# Lab 4 traps

除了上一章实验的页表机制外，中断机制也是组成操作系统不可或缺的部分。本章的实验主要集中于 xv6 的中断机制，通过改进 xv6 的中断机制来学习并应用关于中断的一些概念。

我们先切换到traps分支



## Experiment 1 : 学习 RISC-V 汇编

#### 1. 实验准备

由于本次实验及之后的实验中，会涉及到机器指令级别的操作和调试，故而了解一些关于 RISC-V 汇编的知识将有助于实验的进行。这个实验几乎不需要编写代码，主要内容是观察 一些汇编代码和它们的行为。

**.d .sym .asm .o文件**

这些文件都是在编译和链接过程中生成的中间文件或输出文件：

- .d 文件是依赖文件，包含了源代码文件所依赖的头文件和其他源代码文件的信息。当源代码文件或头文件发生修改时，Make 工具会根据 .d 文件重新生成目标文件。
- .o 文件是目标文件，包含了编译器编译后的机器代码。目标文件是编译和链接的最终结果之一，它包含了程序的代码和数据，在链接的过程中被合并成一个可执行文件。
- .sym 文件是符号表文件，包含了可执行文件中各个函数和全局变量的符号信息。符号表文件通常用于调试和动态链接。
- .asm 文件是反汇编文件，包含了可执行文件或目标文件中的汇编代码。反汇编文件通常用于分析程序的执行流程和优化代码。

在编译 C 语言源代码时，编译器会将源代码文件编译成汇编语言文件（通常扩展名为 .s），然后将汇编语言文件汇编成目标文件（通常扩展名为 .o）。最后，链接器将所有目标文件链接在一起，生成可执行文件。

#### 2. 实验步骤

1. 我们首先需要使用make 将 user/call.c 源文件编译为对应的目标代码，在这个过程中， xv6 的 Makefile 会自动生成反汇编后的代码。编译完成后，打开新生成的user/call.asm，回答 xv6 实验手册中提出的问题。

   ```cmd
   make fs.img && vi user/call.asm
   ```

2. 可以看一下 user/call.c 源文件

   ```c
   #include "kernel/param.h"
   #include "kernel/types.h"
   #include "kernel/stat.h"
   #include "user/user.h"
   
   int g(int x) {
     return x+3;
   }
   
   int f(int x) {
     return g(x);
   }
   
   void main(void) {
     printf("%d %d\n", f(8)+1, 13);
     exit(0);
   }
   ```

3. 把答案存在`answers-traps.txt`下

#### 3. 问题回答

1. **Which registers contain arguments to functions? For example, which register holds 13 in main’s call to printf?**

   通过阅读 call.asm 文件中的 main 函数可知，调用 printf 函数时，13 被寄存器 a2 保存。其中 li a2,13 意思如下：

   ```asm
   void main(void) {
     1c:   1141                    addi    sp,sp,-16
     1e:   e406                    sd      ra,8(sp)
     20:   e022                    sd      s0,0(sp)
     22:   0800                    addi    s0,sp,16
     printf("%d %d\n", f(8)+1, 13);
     24:   4635                    li      a2,13
     26:   45b1                    li      a1,12
     28:   00000517                auipc   a0,0x0
     2c:   7b050513                addi    a0,a0,1968 # 7d8 <malloc+0xe8>
     30:   00000097                auipc   ra,0x0
     34:   608080e7                jalr    1544(ra) # 638 <printf>
     exit(0);
     38:   4501                    li      a0,0
     3a:   00000097                auipc   ra,0x0
     3e:   274080e7                jalr    628(ra) # 2ae <exit>
   ```

   - li：指令助记符，表示将立即数（immediate）加载到寄存器中。
   - a2：寄存器名，表示要将立即数加载到寄存器 a2 中。
   - 13：立即数，表示要加载的整数值。

   **所以答案为**： a1, a2, a3 等通用寄存器；13 被寄存器 a2 保存。

   ```
   'ax' regesters contain arguments to functions, with 'a' stands for 'argument'.
   Register 'a2' holds 13 in main's call to printf.
   ```

2. **Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)**

   通过阅读函数 f 和 g 得知：函数 f 调用函数 g ；函数 g 使传入的参数加 3 后返回。

   所以总结来说，函数 f 就是使传入的参数加 3 后返回。考虑到编译器会进行内联优化，这就意味着一些显而易见的，编译时可以计算的数据会在编译时得出结果，而不是进行函数调用。

   查看 main 函数可以发现，printf 中包含了一个对 f 的调用。

   ```asm
   printf("%d %d\n", f(8)+1, 13);
     24:   4635                    li      a2,13
     26:   45b1                    li      a1,12
     28:   00000517                auipc   a0,0x0
     2c:   7b050513                addi    a0,a0,1968 # 7d8 <malloc+0xe8>
     30:   00000097                auipc   ra,0x0
     34:   608080e7                jalr    1544(ra) # 638 <printf>
   ```

   但是对应的会汇编代码却是直接将 f(8)+1 替换为 12 。这就说明编译器对这个函数调用进行了优化，所以对于 main 函数的汇编代码来说，其并没有调用函数 f 和 g ，而是在运行之前由编译器对其进行了计算。

   **所以答案为**：main 的汇编代码没有调用 f 和 g 函数。编译器对其进行了优化。

   ```
   52: fce080e7                jalr    -50(ra) # 1c <f>
   34: fd0080e7                jalr    -48(ra) # 0 <g>
   ```

3. **At what address is the function printf located?**

   找到printf通过搜索容易得到 printf 函数的位置。

   ```asm
   0000000000000638 <printf>:
   
   void
   printf(const char *fmt, ...)
   ```

   得到其地址在 0x628，**所以答案为**：0x628

   ```
   6c:   9cc080e7                jalr    -1588(ra) # a34 <printf>
   ```

4. **What value is in the register ra just after the jalr to printf in main? **

   auipc 是 RISC-V 汇编指令中的一种，用于将一个 20 位的立即数加上当前指令的地址，得到一个 32 位的地址，并将该地址存储到指定的寄存器中。auipc 指令的全称是 "add upper immediate to PC"，它通常用于实现全局变量、静态变量和函数地址的加载。

   jalr 是 RISC-V 汇编指令中的一种，用于实现函数调用和跳转操作。jalr 指令的全称是 "jump and link register"，它的作用是跳转到指定地址，并将下一条指令的地址（即 PC+4）保存到指定的寄存器中，以便在跳转返回时使用。

   auipc 和 jalr 的配合，可以跳转到任意 32 位的地址。

   ```asm
   # 使用 auipc ra,0x0 将当前程序计数器 pc 的值存入 ra 中
   30:   00000097                auipc   ra,0x0
   # jalr 1528(ra) 跳转到偏移地址 printf 处，也就是 0x628 的位置
   34:   608080e7                jalr    1544(ra) # 638 <printf>
   exit(0);
   38:   4501                    li      a0,0
   ```

   在执行完这句命令之后， 寄存器 ra 的值设置为 pc + 4 ，也就是 return address 返回地址 0x38。

   所以答案为：jalr 指令执行完毕之后，ra 的值为 0x38.

   ```
   0x38
   ```

5. **Run the following code.What is the output?**

   ```asm
   	unsigned int i = 0x00646c72;
   	printf("H%x Wo%s", 57616, &i);
   ```

   [Here's an ASCII table](http://web.cs.mun.ca/~michael/c/ascii-table.html)that maps bytes to characters.

   **The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?**

   [Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](http://www.networksorcery.com/enp/ien/ien137.txt).

   查看在线 C Compiler 的运行结果 [c++ sell](https://cpp.sh/?source=%2F%2F+Example+program %23include+ int+main() { ++unsigned+int+i+%3D+0x00646c72%3B printf("x%3D%d+y%3D%d"%2C+3)%3B return+0%3B })，它打印出了 He110 World。

   首先，57616 转换为 16 进制为 e110，所以格式化描述符 %x 打印出了它的 16 进制值。

   其次，如果在小端（little-endian）处理器中，数据0x00646c72 的高字节存储在内存的高位，那么从内存低位，也就是低字节开始读取，对应的 ASCII 字符为 rld。

   如果在 大端（big-endian）处理器中，数据 0x00646c72 的高字节存储在内存的低位，那么从内存低位，也就是高字节开始读取其 ASCII 码为 dlr。

   所以如果大端序和小端序输出相同的内容 i ，那么在其为大端序的时候，i 的值应该为 0x726c64，这样才能保证从内存低位读取时的输出为 rld 。

   无论 57616 在大端序还是小端序，它的二进制值都为 e110 。大端序和小端序只是改变了多字节数据在内存中的存放方式，并不改变其真正的值的大小，所以 57616 始终打印为二进制 e110 。

   **所以答案为**：如果在大端序，i 的值应该为 0x00646c72 才能保证与小端序输出的内容相同。不用改变 57616 的值。

   ```
   "HE110 World"
   0x00646c73
   No
   ```

6. **In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen**

   通过之前的章节可知，函数的参数是通过寄存器a1, a2 等来传递。如果 prinf 少传递一个参数，那么其仍会从一个确定的寄存器中读取其想要的参数值，但是我们并没有给出这个确定的参数并将其存储在寄存器中，所以函数将从此寄存器中获取到一个随机的不确定的值作为其参数。故而此例中，y=后面的值我们不能够确定，它是一个垃圾值。

   **所以答案为**：y= 之后的值为一个不确定的无效值。

   ```
   The 4-bytes after the address of const 3. This would happen due to undefined memory.
   ```





## Experiment 2 : Backtrace

#### 1. 实验准备

- 因为 gdb 等调试工具很难进行内核态的调试，为了便于调试内核，在内核中实现 backtrace 函数是大有裨益的。我们需要在 kernel/printf.c 实现 backtrace() ，并且在sys_sleep 调用中插入一个 backtrace() 函数，用以打印当时的栈。

- 为了检验实现的正确性，xv6 实验提供了 bttest 程序

- 为了实现 backtrace() 函数，我们需要阅读 RISC-V ABI 文档和一些已经编译好的汇编代码。再次回顾上面的 user/call.asm 中main函数的调用过程：

  ```asm
  void main(void) {
    1c:   1141                    addi    sp,sp,-16
    1e:   e406                    sd      ra,8(sp)
    20:   e022                    sd      s0,0(sp)
    22:   0800                    addi    s0,sp,16
    printf("%d %d\n", f(8)+1, 13);
    24:   4635                    li      a2,13
    26:   45b1                    li      a1,12
    28:   00000517                auipc   a0,0x0
    2c:   7b050513                addi    a0,a0,1968 # 7d8 <malloc+0xe8>
    30:   00000097                auipc   ra,0x0
    34:   608080e7                jalr    1544(ra) # 638 <printf>
    exit(0);
    38:   4501                    li      a0,0
    3a:   00000097                auipc   ra,0x0
    3e:   274080e7                jalr    628(ra) # 2ae <exit>
  ```

  进入 main 函数前，由于栈是从高地址向低地址扩展，所以该程序首先执行 addi sp,sp,-16 将栈向下生长，然后执行两条指令将返回地址 ra 和栈的基地址 s0 存储到栈中。调用其它函数的最开始的几个指令也如出一辙。只是顺带多保存了一些参数到栈中供 printf 函数使用，其 sp 、ra 、s0 的相对位置关系 仍然和文档中一致。

  我们的 backtrace() 函数只需打印每次保存的 ra 即可，然后根据 s0 的值寻找上一个栈帧，然后继续打印保存的 ra ，如此往复直到到达栈底。根据 xv6 实验指导的提醒，我们首先在 kernel/riscv.h 中加入以下内联汇编函数用于读取当前的 s0 寄存器：

  ```c
  static inline uint64
  r_fp()
  {
    uint64 x;
  	// 使用 RISC-V 汇编指令 mv 将 s0 寄存器中的值移动到变量 x 中
    asm volatile("mv %0, s0" : "=r" (x) );
    return x;
  }
  ```

#### 2. 实验步骤

1. 在 kernel/defs.h 中添加该函数声明

   ```c
   void            backtrace(void);
   ```

2. 然后利用上面的思想，在 kernel/printf.c 实现 backtrace():

   ```c
   backtrace(void)=-0=-06
   {
     uint64 fp_address = r_fp();
     while(fp_address != PGROUNDDOWN(fp_address)) {
       // 打印ra
       printf("%p\n", *(uint64*)(fp_address-8));
       // 到下一个帧指针
       fp_address = *(uint64*)(fp_address - 16);
     }
   }
   ```

   其中用到了一些关于指针算数的小技巧，注意，返回地址在帧指针的 -8 偏移量处；前一个帧指针位于当前帧指针的固定偏移量 (-16) 处。每个内核栈由一整个页（4k对齐）组成，所有的栈帧都在同一个页上面。你可以使用PGROUNDDOWN(fp) 来定位帧指针所在的页面，从而确定循环停止的条件,PGROUNDDOWN(fp) 总是表示 fp 所指向的页的起始位置。

3. 实现完成后，在 sys_sleep 调用中插入一个backtrace() 函数：

   ```c
   uint64
   sys_sleep(void)
   {
     int n;
     uint ticks0;
     backtrace();
   ...
   }
   ```

4. 测试

   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-traps backtrace test
   make: 'kernel/kernel' is up to date.
   == Test backtrace test == backtrace test: OK (1.4s)
   ```





## Experiment 3 : Alarm

#### 1. 实验准备

- 为了让一个进程能够在消耗一定的 CPU 时间后被告知，我们需要实现 sigalarm(n, fn) 系统调用。用户程序执行 sigalarm(n, fn) 系统调用后，将会在其每消耗 n 个 tick 的 CPU 时间后被中断并且运行其给定的函数 fn 。这个系统调用对于不希望占用过多 CPU 时间，或希望进行周期性操作的程序来说是及其有用的。此外，实现 sigalarm(n, fn) 系统调用有利于我们学会如何构建用户态的中断和中断处理程序，这样就可以使用类似的方法使用户态程序可以处理缺页等常见的异常。
- 正确实现 sigalarm(n, fn) 系统调用面临以下几个问题：
  1. 如何保存每个进程的 sigalarm 参数；
  2. 如何并更新每个已经消耗的 tick 数；
  3. 如何在已经消耗的 tick 数符合要求时使进程调用中断处理过程 fn ;
  4. 中断处理过程 fn 执行完毕后如何返回。

#### 2. 实验步骤

1. 照例在 user/user.h 和 usys.pl 加入两个系统调用的入口，分别用于设置定时器和从定时器中断处理过程中返回：

   ```c
   //user/user.h
   int sigalarm(int ticks, void (*handler)());
   int sigreturn(void);
   ```

   ```perl
   # user/usys.pl
   entry("sigalarm");
   entry("sigreturn");
   ```

2. 首先解决第一个问题，联想到 Lab System calls 中 trace 的实现，我们需要在每个进程的控制块的数据结构中加入存储 sigalarm(n, fn) 中参数的项，即在 kernel/proc.h 的 struct proc中加入如下的项：

   ```c
   int alarminterval; // sys_sigalarm() alarm interval in ticks
   int alarmticks; // sys_sigalarm() alarm interval in ticks
   void (*alarmhandler)(); // sys_sigalarm() pointer to the alarm handler
   int sigreturned;
   ```

   - alarminterval：sys_sigalarm() 系统调用的定时器间隔，以系统时钟滴答（tick）为单位。一个滴答通常为几毫秒，alarminterval 的值在调用 sys_sigalarm() 时设置。
   - alarmticks：sys_sigalarm() 系统调用的剩余定时器计数器值，以系统时钟滴答为单位。当剩余计数器值为零时，操作系统会向进程发送 SIGALRM 信号。alarmticks 的值在调用 sys_sigalarm() 时设置，通常等于 alarminterval。
   - alarmhandler：指向注册的 SIGALRM 信号处理函数的指针。当操作系统向进程发送 SIGALRM 信号时，进程会执行该函数。
   - sigreturned：用于标识 SIGALRM 信号处理函数是否已经执行完毕。当 SIGALRM 信号处理函数开始执行时，sigreturned 被设置为 0。当 SIGALRM 信号处理函数执行完毕时，sigreturned 被设置为 1。

3. 然后在 kernel/proc.c 的 allocproc(void) 中对这些变量进行初始化：

   ```c
   static struct proc*
   allocproc(void)
   {
   	......
   	// Initialize alarmticks
   	p->alarmticks = 0;
   	p->alarminterval = 0;
   	p->sigreturned = 1;
   	return p;
   }
   ```

4. 初始化后，在调用 sigalarm(n, fn) 系统调用时，执行的 kernel/sysproc.c 中的 sys_sigalarm() 需根据传入的参数设置 struct proc 的对应项：

   ```c
   uint64
   sys_sigalarm(void)
   {
   	int ticks;
   	uint64 handler;
   	struct proc *p = myproc();
   	if(argint(0, &ticks) < 0 || argaddr(1, &handler) < 0)
   	return -1;
   	p->alarminterval = ticks;
   	p->alarmhandler = (void (*)())handler;
   	p->alarmticks = 0;
   	return 0;
   }
   ```

   可以在 sys_sigalarm(void) 加入打印语句用于 debug ，确认参数传入无误后可以着手解决第二个问题。

5. 如何并更新每个已经消耗的 tick 数，我们需要修改定时器中断发生时的行为，为每个有 sigalarm 的进程更新其已经消耗的 tick 数。打开 kernel/trap.c （kernel/trap.c 是 Xv6 内核中的一个文件，它包含了与中断和异常处理相关的代码），在 usertrap()（usertrap() 是在内核中实现用户态异常处理的函数，当用户程序发生异常（例如，页故障、整数溢出等）时，内核会将控制权转移到 usertrap() 函数，由它来处理这个异常）中的判断是否为定时器中断的 if 语句块中加入对应的实现。然后顺手解决第三个问题，判断该进程已经使用的 tick 数是否已经足以触发 alarm ：

   ```c
   ......
   if(which_dev == 2)
   {
   	p->alarmticks += 1;
   	if ((p->alarmticks >= p->alarminterval) && (p->alarminterval > 0))
   	{
   		p->alarmticks = 0;
   		......
   	}
   	yield();
   }
   ......
   ```

6. 若 ticks 达到了预先设定的值，则将计数器清零，并使得用户进程跳至预设的处理过程 fn 处执行。首先我们需要在 struct proc 增设 alarmtrapframe 用于备份进程当前的上下文：

   ```c
   struct trapframe alarmtrapframe; // for saving registers
   ```

   然后，在`usertrap()`中备份当前的上下文，并将用户态的程序计数器设置到 fn 处，然 后完成该定时器调用，返回到用户态：

   ```c
   ......
   if(which_dev == 2)
   {
   	p->alarmticks += 1;
   	if ((p->alarmticks >= p->alarminterval) && (p->alarminterval > 0))
   	{
   		p->alarmticks = 0;
   		if (p->sigreturned == 1)
   		{
   			p->alarmtrapframe = *(p->trapframe);
   			p->trapframe->epc = (uint64)p->alarmhandler;
   			p->sigreturned = 0;
   			usertrapret();
   		}
   	}
   	yield();
   }
   ......
   ```

7. 这样整个触发 sigalarm 的过程便成功完成了。此时还剩最后一个问题没有解决：当传入 的 fn 执行完毕后，如何返回到进程正常的执行过程中。由于之前我们保存了进程执行的上下 文，故而在 fn 调用 sigreturn() 时，我们需要在内核态对应的 sys_sigreturn() 中将备份的上下文恢复，然后返回用户态，该实现依然放在 kernel/sysproc.c 中：

   ```c
   uint64
   sys_sigreturn(void)
   {
     struct proc *p = myproc();
     p->sigreturned = 1;
     *(p->trapframe) = p->alarmtrapframe;
     usertrapret();
     return 0;
   }
   ```

   sys_sigreturn系统调用在执行的特定函数 periodic() 里被显式调用以返回

8. 最后别忘了在 syscall.h 和 syscall.c 中加上刚实现里的两个系统调用的信息，还有 makefile 中填上 alarmtest

9. 本地运行

   ```cmd
   $ alarmtest
   test0 start
   .................................................................alarm!
   test0 passed
   test1 start
   .....alarm!
   .....alarm!
   .....alarm!
   .....alarm!
   .....alarm!
   ....alarm!
   .....alarm!
   .....alarm!
   .....alarm!
   .....alarm!
   test1 passed
   test2 start
   ......................................................................alarm!
   test2 passed
   ```

   测试

   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-traps alarmtest
   make: 'kernel/kernel' is up to date.
   == Test running alarmtest == (3.0s)
   == Test   alarmtest: test0 ==
     alarmtest: test0: OK
   == Test   alarmtest: test1 ==
     alarmtest: test1: OK
   == Test   alarmtest: test2 ==
     alarmtest: test2: OK
   == Test usertests == usertests: OK (72.6s)
   ```

   



## 实验结果

```cmd
yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-traps
make: 'kernel/kernel' is up to date.
== Test answers-traps.txt == answers-traps.txt: OK
== Test backtrace test == backtrace test: OK (1.6s)
== Test running alarmtest == (3.3s)
== Test   alarmtest: test0 ==
  alarmtest: test0: OK
== Test   alarmtest: test1 ==
  alarmtest: test1: OK
== Test   alarmtest: test2 ==
  alarmtest: test2: OK
== Test usertests == usertests: OK (97.8s)
== Test time ==
time: OK
Score: 85/85
```

