# Lab 1 : Xv6 and Unix utilities

## Experiment 1 : 启动`xv6`

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





## Experiment 2 : `sleep`

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

   编译结束后，进行本地测试，输入

   ```cmd
   $ sleep 100 
   // 休眠10秒
   $
   //测试成功
   ```
   
   使用 xv6 实验自带的测评工具测评，退出编译，在终端里输入 ./grade-lab-util sleep ，即可进行自动评测：
   
   ```cmd
   yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util sleep
   make: 'kernel/kernel' is up to date.
   == Test sleep, no arguments == sleep, no arguments: OK (0.8s)
   == Test sleep, returns == sleep, returns: OK (0.9s)
   == Test sleep, makes syscall == sleep, makes syscall: OK (1.1s)
   ```
   
   测试全部通过。





## Experiment 3 : `pingpong`

1. 实验准备

   我们要实现一个名为`pingpong`的实用工具，用以验证`xv6`的进程通信机制。该程序创建一个子进程，并使用管道与子进程通信：进程首先发送一字节的数据给子进程，子进程接收到该数据后，在 shell 中打印`<pid>: received ping` ，其中`pid`为子进程的进程号；随后，向父进程通过管道发送一字节数据，父进程收到数据后在 shell 中打印`<pid>: received pong` ，其中`pid`为父进程的进程号。

   **Hints 1：利用pipe()函数创建管道，pipe()函数接收一个长度为2的数组，数组下标0为读端、1为写端；**
   **Hints 2：利用fork()函数创建新的进程；**
   **Hints 3：利用read、write函数读写管道；**

   我们创建`./user/pingpong.c`并且在`Makefile`中添加该程序，在开始书写代码之前，我们需要了解实现`pingpong`所需要的系统调用：

   1. `pipe`系统调用：创建管道，用于进程间双向通信

      管道：进程间通信的单向通道，由两个文件描述符组成 ( `file descriptors` , `fd` )

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
  $ pingpong
  6: received ping
  5: received pong
  ```

  ```cmd
  yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util pingpong
  make: 'kernel/kernel' is up to date.
  == Test pingpong == pingpong: OK (2.0s)
  ```

- 注意事项

  一定要注意格式，``received ping/pong``没有句号，不按要求输出，测评工具会报错





## Experiment 4 : primes

1. 实验准备

   本次实验我们要利用进程和管道完成一个质数(prime)筛选器，最初的父进程负责产生 2 到 n 的整数，筛选掉其中2的倍数，然后将剩余的整数通过管道写入子进程，依此类推。即每一次筛选，从当前未被筛选的第一个数字开始，排除掉它的倍数，再将剩余的数字写入管道。

   同样，创建 ``uesr/primes.c`` 并将其添加到 `Makefile` 中。

2. 实验步骤

   ```c
   #include "kernel/types.h"
   #include "kernel/stat.h"
   #include "user/user.h"
   
   int main(int argc, char* argv[])
   {
       printf("primes has been called \n");
   
       // 每个管道第一个数必是质数，v是剩下的数
       int first, v;
       // 指向前驱和后继管道的指针
       int* fd1;
       int* fd2;
       // 创建前驱和后继管道
       int first_fd[2];
       int second_fd[2];
   
       pipe(first_fd);
   
       // 创建父进程，将数字2-35输入管道
       if (fork() > 0)
       {
           for (int i = 2; i <= 35; i++)
               write(first_fd[1], &i, sizeof(i));
           close(first_fd[1]);
           int status;
           wait(&status);
       }
       // 对于子进程，循环处理，为筛选质数创建新进程
       else
       {
           fd1 = first_fd;
           fd2 = second_fd;
           while (1)
           {
               pipe(fd2);
               close(fd1[1]);
               if (read(fd1[0], &first, sizeof(first)))
                   printf("prime %d\n", first);
               else
                   break;
               if (fork() > 0)
               {
                   int i = 0;
                   while (read(fd1[0], &v, sizeof(v)))
                   {
                       // 如果可以整除 first，则跳过该值，继续下一个
                       if (v % first == 0)
                           continue;
                       i++;
                       // 如果不可以整除，则将该值写入 fd2 管道中，以供下一个子进程使用。
                       write(fd2[1], &v, sizeof(v));
                   }
                   close(fd1[0]);
                   close(fd2[1]);
                   int status;
                   wait(&status);
                   break;
               }
               // 子进程之间数据的传递
               else
               {
                   close(fd1[0]);
                   int* tmp = fd1;
                   fd1 = fd2;
                   fd2 = tmp;
               }
           }
       }
       //父进程在子进程结束后退出
       exit(0);
   }
   ```

   - 测试

   ```cmd
   $ primes
   prime 2
   prime 3
   prime 5
   prime 7
   prime 11
   prime 13
   prime 17
   prime 19
   prime 23
   prime 29
   prime 31
   ```
   
   ```cmd
   yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util primes
   make: 'kernel/kernel' is up to date.
   == Test primes == primes: OK (0.8s)
   ```





## Experiment 5 : find

1. 实验准备

   编写一个简单版本的UNIX find程序，用于在一个目录树中查找所有具有指定名称的文件

   首先查看`user/ls.c`以了解如何读取目录。 `user/ls.c`中包含一个`fmtname` 函数，用于格式化文件的名称。它通过查找路径中最后一个 `'/'` 后的第一个字符来获取文件的名称部分。如果名称的长度大于等于 `DIRSIZ`，则直接返回名称。否则，将名称拷贝到一个静态字符数组 `buf` 中，并用空格填充剩余的空间，保证输出的名称长度为 `DIRSIZ`。

   - `open`：打开文件并返回一个文件描述符。在`xv6`中，文件描述符是一个非负整数，用于标识已经打开的文件

     输入：

     `char *path` : 要打开的文件的路径

     `int flags` : 打开文件的模式，如只读(0)、只写(1)或读写(2)

     输出：

     返回值为文件描述符，如果出错则返回 -1

   - `close`：关闭一个已经打开的文件描述符，释放相关的资源

     输入：

     `int fd` : 要关闭的文件描述符

     输出：

     0 -成功关闭，-1 -出错

   - `fstat`：获取与文件描述符相关的文件状态信息

     输入：

     `int fd` : 文件描述符

     `struct stat *st` : 指向存储文件状态信息的结构体的指针

     输出：

     0 -成功，状态信息被填充到 `st` 指向的结构体中，-1 -出错

   - `strcmp`：字符串的比较不能使用（==），要使用 `strcmp`。（ C语言中， == 字符串比较的是地址）

   - `memmov`：在内存中移动一块内存数据，这里用于拼接路径

     输入：

     `void *dst` : 目标地址，数据将被复制到此处

     `const void *src` : 源地址，数据将从此处复制

     `size_t n` : 要复制的字节数

     输出：

     返回值是目标地址 `dst`

2. 实验步骤

   为了完成查询，我们的程序要接受两个参数，第一个参数为要查找的目录路径，第二个参数为要查找的文件名，然后在指定目录及其子目录中查找文件名为第二个参数的文件，并输出文件的完整路径。我们写三个函数来完成：

   1. `main` 函数完成参数检查和功能函数的调用。检查命令行参数的数量，如果参数数量小于 3，则输出提示信息并退出程序。输入正确，则调用 `find` 函数进行查找，并最后退出程序。
   2. `match` 函数用于检查给定的路径和名称是否匹配。通过查找路径中最后一个 `'/'` 后的第一个字符来获取文件的名称部分，然后与给定的名称进行比较。如果匹配，则返回 1，否则返回 0。
   3. `find` 函数用于递归地在给定的目录中查找带有特定名称的文件。首先打开目录，并使用 `fstat` 函数获取目录的信息。根据目录的类型进行不同的处理。
      - 如果是文件（`T_FILE`），则使用 `match` 函数检查文件名称是否与给定的名称匹配，如果匹配则打印文件的路径。
      - 如果是目录（`T_DIR`），则遍历目录中的每个文件。首先，检查拼接后的路径长度是否超过了缓冲区 `buf` 的大小。如果超过了，则输出路径过长的提示信息。然后，将路径拷贝到 `buf` 中，并在路径后添加 `'/'`。接下来，循环读取目录中的每个文件信息，注意不要递归到 `'.'` 和`'..'`，跳过当前目录 `'.'` 和上级目录 `'..'`，防止死循环。将文件名拷贝到 `buf` 中，并使用递归调用 `find` 函数在子目录中进行进一步查找。
      - 关闭目录文件描述符。

   ```c
   #include "kernel/types.h"
   #include "kernel/stat.h"
   #include "user/user.h"
   #include "kernel/fs.h"
   
   int match(char* path, char* name)
   {
       // 查找最后一个'/'后的第一个字符
       char* p;
       for (p = path + strlen(path); p >= path && *p != '/'; p--);
       p++;
   
       if (strcmp(p, name) == 0)
           return 1;
       else
           return 0;
   }
   
   void find(char* path, char* name)
   {
       char buf[512], * p;
       int fd;
       struct dirent de;
       struct stat st;
   
       if ((fd = open(path, 0)) < 0)
       {
           fprintf(2, "Error:cannot open %s\n", path);
           return;
       }
   
       if (fstat(fd, &st) < 0)
       {
           fprintf(2, "Error:cannot stat %s\n", path);
           close(fd);
           return;
       }
   
       switch (st.type)
       {
       case T_FILE:
           if (match(path, name))
           {
               printf("%s\n", path);
           }
           break;
   
       case T_DIR:
           if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf)
           {
               printf("Error:path too long\n");
               break;
           }
           strcpy(buf, path);
           p = buf + strlen(buf);
           // 给path后面添加 /
           *p++ = '/';
           while (read(fd, &de, sizeof(de)) == sizeof(de))
           {
               // 如果 inum 为 0，则表示该目录项未被使用，或该目录项已被删除。
               if (de.inum == 0)
                   continue;
               if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                   continue;
               // 把 de.name 接到路径后面
               memmove(p, de.name, DIRSIZ);
               p[DIRSIZ] = 0;
               find(buf, name);
           }
           break;
       }
       close(fd);
   }
   
   int main(int argc, char* argv[])
   {
       if (argc < 3)
       {
           printf("argc is %d and it is less then 2\n", argc);
           exit(1);
       }
       find(argv[1], argv[2]);
       exit(0);
   }
   ```

   - `struct dirent`表示目录项的结构体，用于表示目录中的一个条目，即文件或子目录的基本信息
   - `struct stat` 表示文件状态信息的结构体，用于存储与文件描述符或路径相关的详细信息，例如文件大小、类型、inode编号等

测试：

```cmd
$ mkdir a
$ echo > a/b
$ mkdir c
$ echo > c/b
$ find . b
./a/b
./c/b
```

```cmd
yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util find
make: 'kernel/kernel' is up to date.
== Test find, in current directory == find, in current directory: OK (0.9s)
== Test find, recursive == find, recursive: OK (0.9s)
```





## Experiment 6 : `xargs`

1. 实验准备

   `xargs`从标准输入读取行并对每一行运行一个命令，将该行作为参数提供给命令，常用于批处理文件。

   通过示例理解`xarg`的工作原理：

   ```
   $ echo hello too | xargs echo bye
   bye hello too
   ```

   注意这里的命令是“echo bye”，附加参数是“hello too”，使得命令为“echo bye hello too”，输出“bye hello too”。

   通常有一个最大参数个数，这里利用`kernel/param.h`声明的`MAXARG`，检查声明的数组中参数个数是否超出限制

   - `malloc` : 动态分配一块指定大小的内存。它返回一个指向分配内存的指针
   - `exec` : 用指定的程序替换当前进程的镜像。在调用 `exec` 后，当前进程的代码和数据被新程序的代码和数据所覆盖，新的程序开始执行。如果 `exec` 成功执行，`exec` 之后的代码将不会运行，因为新的程序已经替换了当前进程的代码
   - `free` : 释放动态分配的内存

2. 实验步骤

   1. `main`函数从命令行参数中获取要执行的命令名称和其他参数
   2. `xargs`函数从标准输入中读取一系列字符串，并将这些字符串作为参数传递给指定的命令
   3. `xargs_exec`函数创建一个子进程，并在子进程中调用exec函数来执行指定的命令

   ```c
   #include "kernel/types.h"
   #include "kernel/stat.h"
   #include "kernel/param.h"
   #include "user/user.h"
   
   void xargs_exec(char* program, char** paraments)
   {
       if (fork() > 0) {
           wait(0);
       }
       else {
           if (exec(program, paraments) == -1) {
               fprintf(2, "xargs: Error exec %s\n", program);
           }
       }
   }
   void xargs(char** first_arg, int size, char* program_name)
   {
       char buf[1024];
       char* arg[MAXARG];
       int m = 0;
       while (read(0, buf + m, 1) == 1) {
           // 读取到一个完整的字符串
           if (buf[m] == '\n') {
               // 将该字符串作为参数存储到arg数组中
               arg[0] = program_name;
               for (int i = 0; i < size; ++i) {
                   arg[i + 1] = first_arg[i];
               }
               arg[size + 1] = malloc(sizeof(char) * (m + 1));
               memcpy(arg[size + 1], buf, m + 1);
               // 调用xargs_exec函数执行命令
               xargs_exec(program_name, arg);
               free(arg[size + 1]);
               m = 0;
           }
           else {
               m++;
           }
       }
   }
   int main(int argc, char* argv[])
   {
       char* name = "echo";// 默认执行echo命令，可缺省
       if (argc >= 2) {
           name = argv[1];
       }
       xargs(argv + 1, argc - 1, name);
       exit(0);
   }
   ```

   在`xargs`函数中，首先定义了一个`buf`数组来存储从标准输入中读取到的字符串，以及一个`arg`数组来存储要传递给命令的参数。然后，通过while循环从标准输入中读取一个字符，如果读取到的字符是一个换行符，则表明已经读取到一个完整的字符串，将该字符串存储到`arg`数组中，然后调用`xargs_exec`函数执行命令。如果读取到的字符不是一个换行符，则表明还没有读取到一个完整的字符串，继续读取下一个字符。

   测试

   ```cmd
   $ echo world | xargs echo hello
   hello world
   ```
   
   ```cmd
   yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util xargs
   make: 'kernel/kernel' is up to date.
   == Test xargs == xargs: OK (1.4s)
   ```






## 7 : 实验总结

在完成 Lab Utilities 中的所有实验后，在终端中执行 ./grade-lab-util ，即可对整个实验进行自动评分

```cmd
yuran@silverCraftsman:~/xv6-labs-2021$ ./grade-lab-util
make: 'kernel/kernel' is up to date.
== Test sleep, no arguments == sleep, no arguments: OK (2.1s)
== Test sleep, returns == sleep, returns: OK (0.9s)
== Test sleep, makes syscall == sleep, makes syscall: OK (1.0s)
== Test pingpong == pingpong: OK (1.0s)
== Test primes == primes: OK (1.2s)
== Test find, in current directory == find, in current directory: OK (1.0s)
== Test find, recursive == find, recursive: OK (0.9s)
== Test xargs == xargs: OK (1.2s)
== Test time ==
time: OK
Score: 100/100
```

time是 `MIT 6.S081` 的传统，需要在实验目录下创建一个名为 `time.txt` 文本文件，其中只包含一行，为完成该实验的小时数。