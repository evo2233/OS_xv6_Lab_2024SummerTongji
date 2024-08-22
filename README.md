# OS_xv6_Lab_2024SummerTongji

***

**AUTHOR**: 2250753 宋宇然 Tongji University, 2024 summer vacation OS course design Lab

**PLATFORM**: xv6 —— a simple Unix-like operation system for modern teaching



## Before Labs : Environment Construction

***

#### **STEP1** : 在windows系统上安装`WSL`并且安装`Ubuntu`

**Windows Subsystem for Linux (`WSL`)** : 之所以要下载Linux，是因为`xv6`作为一个简化的操作系统，通常运行在一个模拟器（如 `QEMU`）或虚拟机中，而这个模拟器需要在一个完整的操作系统上运行。在Linux环境中，你可以轻松安装和运行`QEMU`之类的工具，用来启动和调试 `xv6`。

**`Ubuntu`** : 是一种Linux的发行版，`wsl --install` 命令会自动安装`WSL 2`以及默认的 Linux 发行版（通常是 `Ubuntu`）。

[如何使用 `WSL` 在 Windows 上安装 Linux](https://learn.microsoft.com/en-us/windows/wsl/install)

```cmd
C:\Windows\System32>wsl --install
正在安装: 虚拟机平台
已安装 虚拟机平台。
正在安装: 适用于 Linux 的 Windows 子系统
已安装 适用于 Linux 的 Windows 子系统。
正在安装: Ubuntu
已安装 Ubuntu。
请求的操作成功。直到重新启动系统前更改将不会生效。
```

[设置你的 Linux 用户名和密码](https://learn.microsoft.com/en-us/windows/wsl/setup/environment#set-up-your-linux-username-and-password)

```ubuntu
Installing, this may take a few minutes...
wsl: 检测到 localhost 代理配置，但未镜像到 WSL。NAT 模式下的 WSL 不支持 localhost 代理。
Please create a default UNIX user account. The username does not need to match your Windows username.
For more information visit: https://aka.ms/wslusers
Enter new UNIX username: yuran
wsl: 检测到 localhost 代理配置，但未镜像到 WSL。NAT 模式下的 WSL 不支持 localhost 代理。	// VPN导致的，不影响
New password: 111
Retype new password: 111
passwd: password updated successfully
Installation successful!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.153.1-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


This message is shown once a day. To disable it please create the
/home/yuran/.hushlogin file.
```

#### STEP2 : 在`Ubuntu`上执行以下指令

```sudo apt-get update && sudo apt-get upgrade```是检查最新的软件源并且更新软件源

```ubuntu
~$ sudo apt-get update && sudo apt-get upgrade
[sudo] password for yuran:
...
116 upgraded, 0 newly installed, 0 to remove and 4 not upgraded.
Need to get 100 MB of archives.
After this operation, 1596 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
...
```

```sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu```是安装实验所需的各种依赖

```ubuntu
~$ sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
...
0 upgraded, 256 newly installed, 0 to remove and 4 not upgraded.
Need to get 248 MB of archives.
After this operation, 910 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
...
```

#### STEP3 : 克隆`xv6`实验所需源码

输入```git clone git://g.csail.mit.edu/xv6-labs-2021```

```ubuntu
~$ git clone git://g.csail.mit.edu/xv6-labs-2021
Cloning into 'xv6-labs-2021'...
remote: Enumerating objects: 7051, done.
remote: Counting objects: 100% (7051/7051), done.
remote: Compressing objects: 100% (3423/3423), done.
remote: Total 7051 (delta 3702), reused 6830 (delta 3600), pack-reused 0
Receiving objects: 100% (7051/7051), 17.20 MiB | 4.98 MiB/s, done.
Resolving deltas: 100% (3702/3702), done.
warning: remote HEAD refers to nonexistent ref, unable to checkout.	// branch问题，下载没失败
```

#### STEP4 : 切换好目录并切换到实验分支进行编译，进入操作系统

依次输入```cd xv6-labs-2021/```、```git checkout util```、```make qemu```

```ubuntu
~$ cd xv6-labs-2021/
~/xv6-labs-2021$ git checkout util
Branch 'util' set up to track remote branch 'util' from 'origin'.
Switched to a new branch 'util'
~/xv6-labs-2021$ make qemu
...
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
```

到此成功启动`xv6`，可以使用 ls 指令列出根目录下文件，然后试着执行一些内置的程序，例如执行`cat README`，即可查看`README`文件的内容。



## Lab 1 : xv6 and Unix utilities

#### STEP1 : 熟悉`xv6`

1. **查看代码** : ``ls``、``cat``、``less``

​	我们实验中使用的`xv6`安装在Linux子系统中，众所周知，Linux没有图形化界面，所以要查看文件代码需要在命令行中进行。你可以通过``cd xv6-labs-2021/user``进入user文件夹（`../`是返回上一级），通过``ls``命令查看其中文件，许多是C语言的文件，与我们的实验代码一样。你可以用``cat``命令查看其中的内容，下面我们查看一下`cat.c`的代码，了解一下代码风格。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

char buf[512];

void
cat(int fd)
{
  int n;

  while((n = read(fd, buf, sizeof(buf))) > 0) {
    if (write(1, buf, n) != n) {
      fprintf(2, "cat: write error\n");
      exit(1);
    }
  }
  if(n < 0){
    fprintf(2, "cat: read error\n");
    exit(1);
  }
}

int
main(int argc, char *argv[])
{
  int fd, i;

  if(argc <= 1){
    cat(0);
    exit(0);
  }

  for(i = 1; i < argc; i++){
    if((fd = open(argv[i], 0)) < 0){
      fprintf(2, "cat: cannot open %s\n", argv[i]);
      exit(1);
    }
    cat(fd);
    close(fd);
  }
  exit(0);
}
```

观察上述代码，发现：

- 头文件引用自`xv6`内核，其中：
  	- `types.h`：定义了`xv6`操作系统中使用的基本数据类型和常量，例如`int、uint、uchar、ushort`等；
  	- `stat.h`：定义了`xv6`操作系统中的文件状态结构体和文件权限常量，例如`struct stat、S_IFMT、S_IFDIR`等；
  	- `user.h`：定义了`xv6`操作系统中的用户空间系统调用接口，包括文件操作、进程管理、内存管理、信号处理等。通过包含`user.h`头文件，用户程序可以调用`xv6`操作系统提供的各种系统调用函数，从而实现各种功能。
- 结束函数不使用`return`，而是调用`exit`。这是因为在用户程序中，如果仅仅使用`return`，并不会通知操作系统该进程已经完成，操作系统也无法进行资源回收和进程调度。`exit` 是一个系统调用（或库函数），用于明确地告诉操作系统当前进程已经结束，并传递一个退出状态码。操作系统接收到这个信号后，会进行相应的资源回收和进程调度工作。

补充：在文本过长时，通过``less``命令，你可以打开一个**分页查看器**，分页阅读文本（使用空格键向下翻页，使用 b 键向上翻页，按下 q 键退出）

2. **上传代码**：``>``、``mv``、``vim``

直接使用重定向符 `>` 创建空文件或重定向输出到文件。

```cmd
> filename.txt
```

如果位置或命名错误，用``mv``移动位置或重命名。也可以用``rm``删除

```cmd
mv old new
```

`Vim`是一个**文本编辑器**，可以通过``vim``命令，在`Vim`中打开文件。

进入`Vim`后，默认是**正常模式**，在这个模式下无法直接编辑文件内容。你可以通过以下按键进入**插入模式**：

- **`i`**：在当前光标位置**前**插入文字。
- **`I`**：在当前行的行首插入文字。
- **`a`**：在当前光标位置**后**插入文字。
- **`A`**：在当前行的行尾插入文字。
- **`o`**：在当前行的**下方**新建一行。
- **`O`**：在当前行的**上方**新建一行。

当你完成编辑后，按下`Esc`键退出插入模式，返回到正常模式。

在正常模式下，输入冒号`:`字符，会在命令行底部出现一个冒号提示符。输入`:q`，然后按下回车键，执行不保存退出命令。

如果您希望保存更改，输入`:wq`以保存更改。

#### STEP2 : 实现`sleep`命令

1. 实验准备

   本实验的任务是，编写声明于``user.h``的用户态函数。它负责调用封装了系统调用的函数，后者与内核函数交互，完成定时休眠。

   我们先在``user/user.h``中找到系统调用函数sleep的声明：

   ```c
   // system calls
   ...
   int sleep(int);
   ```

   再在``user/user.S``中查看定义系统调用的汇编语句：

   ```
   .global sleep
   sleep:
    li a7, SYS_sleep	// 加载立即数，存入a7寄存器
    ecall
    ret
   ```

   上面这段代码调用的``SYS_sleep``就是内核函数，位于``kernel/sysproc.c``中，查看后发现它实现了休眠的逻辑：

   ```c
   sys_sleep(void)
   {
     int n;		// 休眠的滴答数
     uint ticks0;	// 休眠开始时间
   
     if(argint(0, &n) < 0)
       return -1;
     acquire(&tickslock);	// ticks获取锁
     ticks0 = ticks;		// 获取休眠开始时间
     while(ticks - ticks0 < n){
       if(myproc()->killed){	// 检查进程是否被kill
         release(&tickslock);
         return -1;
       }
       sleep(&ticks, &tickslock);	// sleep的调用	
     }
     release(&tickslock);
     return 0;
   }
   ```

   我们要写的函数，应该位于``user/sleep.c``中，这个文件的`main`函数，从用户调用接收命令行字符串。调用`user.S`中声明的全局`sleep`函数，同时传递一个`int`类型的参数。

2. 实验步骤

   代码编写如下：

   ```c
   #include "kernel/types.h"
   #include "user/user.h"
   
   int main(int argc, char* argv[])
   {
       if (argc != 2) {
           fprintf(2, "Usage: sleep <times>\n");
       }
       int time = atoi(*++argv);
       if (sleep(time) != 0) {
           fprintf(2, "Error in sleep sys_call\n");
       }
       exit(0);
   }
   ```

   1. **`argc`** 整数参数，其值是命令行参数的数量+1。通过检查 `argc` 的值，程序判断是否提供了足够的命令行参数。

   2. **`atoi`** 将命令行参数转换为整数的系统函数。

   3. **`++argv`** 是将指针移动到`argv[1]`，也就是用户提供的时间参数。

      **`*argv`** 则是获取这个指针所指向的字符串值。

   利用`vim`编辑器，将代码输入``user/sleep.c``中。

   然后我们要将写好的源文件放入`Makefile`文件的`UPROGS`环境变量中。

   ```makefile
   UPROGS=\
   	$U/_sleep\
   ```

   保存后在终端里执行QEMU的make命令`make qemu`，进行编译。

   使用 xv6 实验自带的测评工具测评，在终端里输入 ./grade-lab-util sleep ，即可进行自动评测：

   ```cmd
   yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util sleep
   make: 'kernel/kernel' is up to date.
   == Test sleep, no arguments == sleep, no arguments: OK (0.8s)
   == Test sleep, returns == sleep, returns: OK (0.9s)
   == Test sleep, makes syscall == sleep, makes syscall: OK (1.1s)
   ```

   测试全部通过即可，也可以自己通过`sleep`命令测试。

#### STEP3 : 实现`pingpong`命令

1. 实验准备

   我们要实现一个名为`pingpong`的实用工具，用以验证`xv6`的进程通信机制。该程序创建一个子进程，并使用管道与子进程通信：进程首先发送一字节的数据给子进程，子进程接收到该数据后，在 shell 中打印`<pid>: received ping` ，其中`pid`为子进程的进程号；随后，向父进程通过管道发送一字节数据，父进程收到数据后在 shell 中打印`<pid>: received pong` ，其中`pid`为父进程的进程号。

   我们创建`./user/pingpong.c`并且在`Makefile`中添加该程序，在开始书写代码之前，我们需要了解实现`pingpong`所需要的系统调用：

   1. `pipe`系统调用：创建管道，用于进程间双向通信

      **输入**：`fd[2]`二元数组

      - `fd[0]`-管道读取端
      - `fd[1]`-管道写入端

      ```c
      char fd[2];
      pipe(fd);
      ```

      **输出**：

      - 返回0-成功
      - 返回-1-失败

   2. `fork`系统调用：创建子进程，由父进程调用，作为其副本

      **输入**：无

      **输出**：

      - 父进程中-返回子进程PID(一个正整数)
      - 子进程中-返回0
      - 失败-返回-1

      ```c
      int pid = fork();	// pid>0\pid==0\pid==-1
      ```

   3. `read`系统调用：读取管道中的数据

      **输入**:

      - `fd`: 文件描述符，指示要从哪个文件或管道读取数据。
      - `buffer`: 存储读取数据的缓冲区指针。
      - `count`: 要读取的字节数。

      ```c
      read(fd[0], buffer, 1)
      ```

      **输出**：

      - 成功-返回读取的字节数
      - 失败-返回-1

   4. `write`系统调用：写管道

      **输入**：

      - `fd`: 文件描述符，指示要写入哪个文件或管道。

      - `buffer`: 包含要写入数据的缓冲区指针。

      - `count`: 要写入的字节数。

      ```c
      write(fd[1], buffer, 1)
      ```

      **输出**：

      - 成功-返回写入的字节数

      - 失败-返回-1

   5. `getpid`系统调用：获取进程号

      ```c
      getpid()
      ```

   此外，`./user/user.h`提供了`int fprintf(FILE *stream, const char *format, ...);`函数，用于 shell 中的格式化输出。

2. 实验步骤

- 创建并打开`user/pingpong.c`文件

- 在其中编写代码并保存：

  ```c
  #include "kernel/types.h"
  #include "kernel/stat.h"
  #include "user/user.h"
  
  int main(int argc, char* argv[])
  {
  	int fd[2];
  	if (pipe(fd) == -1) {
  		fprintf(2, "Error: pipe(fd) error.\n");
  	}
  
  	int pid = fork();
  	if (pid == 0) {
  		char buffer[1];
  		read(fd[0], buffer, 1);
  		close(fd[0]);
  		fprintf(1, "%d: received ping\n", getpid());
  		write(fd[1], buffer, 1);
  	}
  	else if (pid > 0) {
  		char buffer[1];
  		buffer[0] = 'a';
  		write(fd[1], buffer, 1);
  		close(fd[1]);
  		read(fd[0], buffer, 1);
  		fprintf(1, "%d: received pong\n", getpid());
  	}
  	else {
  		fprintf(2, "Error: fork() error.\n");
  		exit(1);
  	}
  	exit(0);
  }
  ```

- 测试

  ```cmd
  yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util pingpong
  make: 'kernel/kernel' is up to date.
  == Test pingpong == pingpong: OK (2.0s)
  ```

- 注意事项

  一定要注意格式，``received ping/pong``没有句号，不按要求输出，会报错

#### STEP4 : primes筛选器

