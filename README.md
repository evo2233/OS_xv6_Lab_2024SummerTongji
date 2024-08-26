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

