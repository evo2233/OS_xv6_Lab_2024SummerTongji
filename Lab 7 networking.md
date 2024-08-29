# Lab 7 networking

驱动程序是操作系统的重要组成部分，负责沟通操作系统和实际工作的硬件。在本次实 验中，我们会在 xv6 中为一个现实中广泛使用的硬件：Intel E1000 网卡，编写驱动程序，从而 体会编写驱动程序的一般步骤。

首先我们切换到 net 分支



## Experiment 1 : 实现 Intel E1000 网卡的驱动

#### 1. 实验准备

Intel E1000 网卡是一类常见的千兆以太网卡，广泛用于各类个人电脑和服务器中。由于 其支持较为完善，且文档齐全，故而在我们的 qemu 中也有软件模拟的 E1000 设备，可供 xv6实验使用。

在开始编写驱动程序前，我们需要获取 Intel 提供的关于 E1000 网卡的驱动的开发者文档 Intel E1000 Software Developer’s Manual 1 ，其中包含了关于该网卡硬件特性和工作机制的说明。根据 xv6 的实验手册，我们主要需要关注以下内容：

1. Section 2 ：关于 E1000 的基本信息
2. Section 3.2 ：接收数据包的简介
3. Section 3.3 和 Section 3.4 ：发送数据包的简介
4. Section 13 ：E1000 使用到的寄存器
5. Section 14 ：E1000 的设备初始化过程

大致浏览过上面的内容后，我们主要需要实现 kernel/e1000.c 中的两个函数：用于发送 数据包的 e1000_transmit() 和用于接收数据包的 e1000_recv() 。

#### 2. 实验步骤

1. 用于初始化设备的 e1000_init() 已经被实现好了，我们主要关注的是其涉及到的一些数据结构：

   ```perl
   #define TX_RING_SIZE 16
   static struct tx_desc tx_ring[TX_RING_SIZE] __attribute__((aligned(16)));
   static struct mbuf *tx_mbufs[TX_RING_SIZE];
   
   #define RX_RING_SIZE 16
   static struct rx_desc rx_ring[RX_RING_SIZE] __attribute__((aligned(16)));
   static struct mbuf *rx_mbufs[RX_RING_SIZE];
   
   // remember where the e1000's registers live.
   static volatile uint32 *regs;
   
   struct spinlock e1000_lock;
   ```

   其中最重要的是两个环形缓冲区：tx_ring 和 rx_ring 。根据 Intel E1000 Software Developer’s Manual 上的描述，我们只需要将需要发送的数据包放入环形缓冲区中，设置好对应的参数并更新管理缓冲区的寄存器，即可视为完成了数据包的发送。

2. 此后网卡的硬件会自动在合适的时间将我们放入的数据包按照配置发送出去，在 kernel/e1000.c 中的实现如下：

   ```c
   int
   e1000_transmit(struct mbuf *m)
   {
     //
     // Your code here.
     acquire(&e1000_lock);
     //printf("e1000_transmit: called mbuf=%p\n",m);
     uint32 idx = regs[E1000_TDT];
     if (tx_ring[idx].status != E1000_TXD_STAT_DD)
     {
       printf("e1000_transmit: tx queue full\n");
       // __sync_synchronize();
       release(&e1000_lock);
       return -1;
     } else {
       if (tx_mbufs[idx] != 0)
       {
         mbuffree(tx_mbufs[idx]);
       }
       tx_ring[idx].addr = (uint64) m->head;
       tx_ring[idx].length = (uint16) m->len;
       tx_ring[idx].cso = 0;
       tx_ring[idx].css = 0;
       tx_ring[idx].cmd = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;
       tx_mbufs[idx] = m;
       regs[E1000_TDT] = (regs[E1000_TDT] + 1) % TX_RING_SIZE;
     }
     // __sync_synchronize();
     release(&e1000_lock);
     return 0;
   }
   ```

   该函数的具体实现流程如下：

   1. 首先获取全局锁e1000_lock，保证同一时刻只有一个线程可以执行该函数，避免多个线程同时操作网卡产生竞态条件。
   2. 打印调试信息，输出传入函数的mbuf结构体的地址。
   3. 从网络发送环形缓冲区的队列头部获取当前可用于发送的缓冲区的下标idx。（E1000_TDT代表了发送环形缓冲区队列的队列尾指针。）
   4. 如果该缓冲区的状态为非“已完成”（即E1000_TXD_STAT_DD），则表示该缓冲区已被占用，等待缓冲区空闲后再进行发送。函数返回-1，表示发送失败。否则，继续下一步操作。
   5. 如果该缓冲区的状态为“已完成”，则表示该缓冲区可以用于发送数据。首先释放当前占用缓冲区的mbuf结构体，然后将传入函数的mbuf结构体中的数据复制到该缓冲区中，设置缓冲区的相关参数，如缓冲区地址、长度等。并将该缓冲区的状态设置为“未完成”。最后，将传入函数的mbuf结构体的指针存储在tx_mbufs数组中，以便在发送完成后释放该结构体。同时，更新发送环形缓冲区的队列头部的下标regs[E1000_TDT]，表示已经使用了该缓冲区进行数据发送。
   6. 释放全局锁e1000_lock，函数调用结束。
   7. 返回0，表示数据发送成功。

   在该函数中，使用了全局锁e1000_lock来保证多个线程之间的互斥访问，避免产生竞态条件。同时，使用了环形缓冲区来实现数据发送的队列化，以提高数据发送的效率和保证时序性。

3. 对于 e1000_recv() ，也是同样的，只是在将数据包拷贝出 rx_ring 时，需要使用合适的函数。根据 xv6 实验手册的提示，我们无需自己实现该拷贝过程，而只需要调用 net.c 中的net_rx() 即可。注意当网卡硬件产生一次中断后，我们的中断处理过程 e1000_intr 会执行regs[E1000_ICR] = 0xffffffff 从而告知网卡我们已经完成了这次中断中所有数据包的处理，故而我们的 e1000_recv() 需要将 rx_ring 中的所有内容都拷贝出去，实现如下：

   ```c
   extern void net_rx(struct mbuf *);
   static void
   e1000_recv(void)
   {
     //
     // Your code here.
     uint32 idx = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
     struct rx_desc* dest = &rx_ring[idx];
     while (rx_ring[idx].status & E1000_RXD_STAT_DD)
     {
       acquire(&e1000_lock);
       struct mbuf *buf = rx_mbufs[idx];
       mbufput(buf, dest->length);
       if (!(rx_mbufs[idx] = mbufalloc(0)))
   	    panic("mbuf alloc failed");
       dest->addr = (uint64)rx_mbufs[idx]->head;
       dest->status = 0;
       regs[E1000_RDT] = idx;
       // __sync_synchronize();
       release(&e1000_lock);
       net_rx(buf);
       idx = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
       dest = &rx_ring[idx];
     }
   }
   ```

   首先，该代码获取当前可用于接收的缓冲区下标，即将接收指针递增1并对缓冲区大小取模的结果。然后，它获取该缓冲区对应的接收描述符（rx_desc结构体），用于存储接收到的数据包的信息。

   接下来，该函数进入一个循环，不断检查当前缓冲区是否接收到了数据包。如果当前缓冲区接收到了数据包，那么函数将获取该数据包所对应的内存缓冲区（mbuf结构体），并将数据包的长度存储到接收描述符中。然后，函数会为下一个可用于接收的缓冲区分配一个新的内存缓冲区，并将该缓冲区的地址存储到接收描述符中。最后，函数将接收指针指向下一个可用于接收的缓冲区，并调用net_rx()函数将接收到的数据包传递给网络协议栈处理。

4. 这里同样需要加锁保证互斥，测试结果如下：

   ```cmd
   ping无应答
   ```

   



## 实验结果

```cmd
Failed
```

