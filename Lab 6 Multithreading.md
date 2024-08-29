# Lab 6 Multithreading

用户态线程作为轻量级的进程，相比于进程有着更加方便的通信机制和更加灵活的使用 方法。本次实验的主题是用户态线程，我们将主要在用户态进行线程相关的实验。

首先切换到 thread 分支



## Experiment 1 : Uthread: switching between threads

#### 1. 实验准备

- 在本实验中，我们需要完成一个用户态线程库中的功能。xv6 为该实验提供了基本的代码： user/uthread.c 和 user/uthread_switch.S 。我们需要在 user/uthread.c 中实现 thread_create() 和 thread_schedule() ，并且在 user/uthread_switch.S 中实现 thread_switch用于切换上下文

#### 2. 实验步骤

1. 首先我们查看 user/uthread.c 中关于线程的一些数据结构：

   ```c
   /* Possible states of a thread: */
   #define FREE        0x0
   #define RUNNING     0x1
   #define RUNNABLE    0x2
   
   #define STACK_SIZE  8192
   #define MAX_THREAD  4
   
   
   struct thread {
     char       stack[STACK_SIZE]; /* the thread's stack */
     int        state;             /* FREE, RUNNING, RUNNABLE */
   };
   struct thread all_thread[MAX_THREAD];
   struct thread *current_thread;
   ```

2. 线程的数据结构十分简洁，struct thread 中，一个字节数组用作线程的栈，一个整数用于表示线程的状态。不难发现我们还需要增加一个数据结构用于保存每个线程的上下文，故参照内核中关于进程上下文的代码，增加以下内容并把它加到上述 thread结构体中：

   ```c
   // Saved registers for user context switches.
   struct context {
     uint64 ra;
     uint64 sp;
     // callee-saved
     uint64 s0;
     uint64 s1;
     uint64 s2;
     uint64 s3;
     uint64 s4;
     uint64 s5;
     uint64 s6;
     uint64 s7;
     uint64 s8;
     uint64 s9;
     uint64 s10;
     uint64 s11;
   };
   ```

3. 参考 xv6 实验手册的提示，除了 sp 、s0 和 ra 寄存器，我们只需要保存 callee-saved 寄存器，因此构造了上面的 struct context 结构体。有了该结构体，我们仿照 kernel/trampoline.S的结构，按照 struct context 各项在内存中的位置，在 user/uthread_switch.S 中加入如下的代码：

   ```c
   thread_switch:
   	/* YOUR CODE HERE */
   	
   	/* Save registers */
   	sd ra, 0(a0)
   	sd sp, 8(a0)
   	sd s0, 16(a0)
   	sd s1, 24(a0)
   	sd s2, 32(a0)
   	sd s3, 40(a0)
   	sd s4, 48(a0)
   	sd s5, 56(a0)
   	sd s6, 64(a0)
   	sd s7, 72(a0)
   	sd s8, 80(a0)
   	sd s9, 88(a0)
   	sd s10, 96(a0)
   	sd s11, 104(a0)
   	/* Restore registers */
   	ld ra, 0(a1)
   	ld sp, 8(a1)
   	ld s0, 16(a1)
   	ld s1, 24(a1)
   	ld s2, 32(a1)
   	ld s3, 40(a1)
   	ld s4, 48(a1)
   	ld s5, 56(a1)
   	ld s6, 64(a1)
   	ld s7, 72(a1)
   	ld s8, 80(a1)
   	ld s9, 88(a1)
   	ld s10, 96(a1)
   	ld s11, 104(a1)
   
   	ret    /* return to ra */
   ```

   以上函数的实现过程如下：

   1. 首先，在保存寄存器状态之前，需要将保存寄存器的内存地址传递给该函数，这里假设该地址保存在寄存器a0中，恢复寄存器的地址保存在寄存器a1中。
   2. 接下来，使用sd指令将当前线程的寄存器状态保存到内存中。具体来说，将返回地址寄存器ra、栈指针寄存器sp和调用者保存寄存器s0s11的值分别保存到a0指向的内存区域的偏移量为0-104的位置。
   3. 然后，使用ld指令将要切换到的线程的寄存器状态从内存中恢复。具体来说，将返回地址寄存器ra、栈指针寄存器sp和调用者保存寄存器s0s11的值分别从a1指向的内存区域的偏移量为0-104的位置读取出来。
   4. 最后，使用ret指令返回到返回地址寄存器ra指向的位置，即切换到另一个线程的代码中执行。

   这样就完成了上下文切换的功能，接下来需要完成创建线程和调度线程的部分。

4. 创建线程时，我们需要将线程的栈设置好，并且需要保证在线程被调度运行时能够将 pc 跳转到正确的位置。上面的 thread_switch 在保存第一个进程的上下文后会加载第二个进程的上下文，然后跳至刚刚加载的 ra 地址处开始执行，故而我们在创建进程时只需将 ra 设为我们所要执行的线程的函数地址即可。于是 thread_create() 的实现如下：

   ```c
   .....	
   	// YOUR CODE HERE
     t->context.ra = (uint64)func;
     t->context.sp = (uint64)&t->stack[STACK_SIZE]
   ```

5. 类似的，在调度线程时，我们选中下一个可运行的线程后，使用 thread_switch 切换上下文即可，实现如下：

   ```c
   void
   thread_schedule(void)
   {
   	......
   	/* YOUR CODE HERE
   	* Invoke thread_switch to switch from t to next_thread:
   	* thread_switch(??, ??);
   	*/
   	// t->state = RUNNABLE;
   	thread_switch((uint64)&t->context, (uint64)&current_thread->context);
   	......
   }
   ```

6. 这样便完成了线程库的基本功能。编译运行 xv6 ，然后执行 uthread ，进行本地测试

   ```cmd
   $ uthread
   thread_a started
   thread_b started
   thread_c started
   thread_c 0
   thread_a 0
   thread_b 0
   ...
   thread_c 99
   thread_a 99
   thread_b 99
   thread_c: exit after 100
   thread_a: exit after 100
   thread_b: exit after 100
   thread_schedule: no runnable threads
   ```





## Experiment 2 : Using threads

#### 1. 实验准备

- 本部分的第二个实验不涉及到 xv6 ，而是在常规的拥有多核的 Linux 机器下进行（需要将我们的虚拟机的 CPU 数量调整为大于 1 的值），使用著名的 pthread 库来研究多线程中的一些问题。本实验中，我们需要修改 notxv6/ph.c ，使之在多线程读写一个哈希表的情况下能够产生正确的结果。
- 哈希表是一种常见的数据结构，用于存储键值对，而 ph.c 中实现了哈希表的一些基本操作，如插入、查找和删除等。在多线程环境下进行哈希表操作时，可能会出现一些线程安全的问题，例如竞态条件或死锁等，因此需要使用同步机制来保证线程安全。
- 在该实验中，需要使用pthread库提供的同步机制（例如互斥锁、条件变量等）来保证多线程对哈希表的操作是线程安全的，并且确保多个线程能够正确地读写哈希表。通过这个实验，可以加深对多线程编程的理解，了解多线程编程中可能出现的问题，并学习如何使用同步机制来解决这些问题。

#### 2. 实验步骤

1. 首先运行 make ph 编译 notxv6/ph.c ，运行 ./ph 1 ，可以获得类似下面输出：

   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ make ph
   gcc -o ph -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/ph.c -pthread
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./ph 1
   100000 puts, 4.106 seconds, 24353 puts/second
   0: 0 keys missing
   100000 gets, 4.206 seconds, 23778 gets/second
   ```

2. 不难发现，写入哈希表的数据被完整地读出，没有遗漏。然后运行 ./ph 2 ，则输出如下：

   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./ph 2
   100000 puts, 2.595 seconds, 38532 puts/second
   0: 16906 keys missing
   1: 16906 keys missing
   200000 gets, 5.012 seconds, 39904 gets/second
   ```

   在多线程同时读写的情况下，部分数据由于竞争访问，一些数据没有被正确写入到哈希表中，因而我们需要解决这个问题。一个常规的解决方案是给共享数据结构加上锁，获得锁的线程才可以写该数据结构。

3. 下面是这个哈希表数据结构的组成：

   ```c
   #define NBUCKET 5
   #define NKEYS 100000
   
   struct entry {
     int key;
     int value;
     struct entry *next;
   };
   struct entry *table[NBUCKET];
   int keys[NKEYS];
   ```

   可以看到哈希表共有 5 个 bucket，每个 bucket 都是一个由 entry 组成的链表

   而哈希表的插入操作也很直观：

   ```c
   static void
   insert(int key, int value, struct entry **p, struct entry *n)
   {
     struct entry *e = malloc(sizeof(struct entry));
     e->key = key;
     e->value = value;
     e->next = n;
     *p = e;
   }
   ```

4. 但上面这段简单的代码也是多线程操作哈希表时出错的根源，假设某个线程调用了 insert 但没有返回，此时另一个线程调用 insert，它们的第四个参数 n（bucket 的链表头）如果值相同，就会发生漏插入键值对的现象。

   为了避免上面的错误，只需要在 input 调用 insert 的部分加锁即可。

   ```c
   pthread_mutex_t locks[NBUCKET];
   
   static
   void put(int key, int value)
   {
     int i = key % NBUCKET;
   
     // is the key already present?
     struct entry *e = 0;
     for (e = table[i]; e != 0; e = e->next) {
       if (e->key == key)
         break;
     }
     if(e){
       // update the existing key.
       e->value = value;
     } else {
       // the key is new.
       pthread_mutex_lock(&locks[i]);
       insert(key, value, &table[i], table[i]);
       pthread_mutex_unlock(&lock[i]);
     }
   }
   ```

5. 测试

   再使用 make ph 编译 notxv6/ph.c ，运行 ./ph 2 ，则发现没有读出的数据缺失，如下

   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./ph 2
   100000 puts, 2.569 seconds, 38922 puts/second
   1: 0 keys missing
   0: 0 keys missing
   200000 gets, 4.988 seconds, 40097 gets/second
   ```





## Experiment 3 : Barrier

#### 1. 实验准备

- 除了锁以外，线程屏障 barrier 也是线程进行同步的重要机制。线程屏障 barrier 可以看成线程的检查点，即某个参与 barrier 的线程先执行到 barrier() 语句处时，其需要等待其它尚未到达 barrier() 处的线程；当所有参与线程屏障的线程都到达 barrier() 语句处时，所有参与线程屏障的线程都继续运行。

- 首先运行 make barrier 编译 notxv6/barrier.c ，运行 ./barrier 2 ，可以获得类似下面输出：

  ```cmd
  yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ make barrier
  gcc -o barrier -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/barrier.c -pthread
  yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./barrier 2
  barrier: notxv6/barrier.c:45: thread: Assertion `i == t' failed.
  barrier: notxv6/barrier.c:45: thread: Assertion `i == t' failed.
  Aborted
  ```

  说明该 barrier() 并未正确实现。

#### 2. 实验步骤

1. 根据 xv6 实验手册中的提示，可以使用条件变量来实现 barrier 机制。首先查看 barrier.c 中相关的数据结构：

   ```c
   struct barrier {
     pthread_mutex_t barrier_mutex;
     pthread_cond_t barrier_cond;
     int nthread;      // Number of threads that have reached this round of the barrier
     int round;     // Barrier round
   } bstate;
   ```

2. 为实现线程屏障，需要维护一个互斥锁、一个条件变量、用以记录到达线程屏障的线程数的整数和记录线程屏障轮数的整数。在初始化用的 barrier_init() 中，互斥锁、条件变量及nthread 被初始化。此后在某个线程到达 barrier() 时，需要获取互斥锁进而修改 nthread。当 nthread 与预定的值相等时，将 nthread 清零，轮数加一，并唤醒所有等待中的线程。最后不要忘记在 barrier() 中释放互斥锁。实现如下所示：

   ```c
   static void 
   barrier()
   {
     // YOUR CODE HERE
     //
     // Block until all threads have called barrier() and
     // then increment bstate.round.
     //
     pthread_mutex_lock(&bstate.barrier_mutex);
     bstate.nthread ++;
     if (bstate.nthread == nthread)
     {
       bstate.round++;
       bstate.nthread = 0;
       pthread_cond_broadcast(&bstate.barrier_cond);
     } else {
       pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
     }
     pthread_mutex_unlock(&bstate.barrier_mutex);
   }
   ```

3. 此时使用 make barrier 编译 notxv6/barrier.c ，运行 ./barrier 2 ，则通过测试，如下：

   ```cmd
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ make barrier
   gcc -o barrier -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/barrier.c -pthread
   yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./barrier 2
   OK; passed
   ```





## 实验结果

```cmd
yuran@silverCraftsman:~/OS_xv6_Lab_2024SummerTongji$ ./grade-lab-thread
make: 'kernel/kernel' is up to date.
== Test uthread == uthread: OK (1.8s)
== Test answers-thread.txt == answers-thread.txt: OK
== Test ph_safe == make: 'ph' is up to date.
ph_safe: OK (7.8s)
== Test ph_fast == make: 'ph' is up to date.
ph_fast: OK (19.1s)
== Test barrier == make: 'barrier' is up to date.
barrier: OK (3.3s)
== Test time ==
time: OK
Score: 60/60
```



