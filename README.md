# OS_xv6_Lab_2024SummerTongji

***

**AUTHOR**: 2250753 宋宇然 Tongji University, 2024 summer vacation OS course design Lab

**PLATFORM**: xv6 —— a simple Unix-like operation system for modern teaching



## Chapter 0 : Environment Construction

***

#### **STEP1** : 在windows系统上安装WSL并且安装Ubuntu

**Windows Subsystem for Linux (WSL)** : 之所以要下载Linux，是因为xv6作为一个简化的操作系统，通常运行在一个模拟器（如 QEMU）或虚拟机中，而这个模拟器需要在一个完整的操作系统上运行。在 Linux 环境中，你可以轻松安装和运行 QEMU 之类的工具，用来启动和调试 xv6。

[如何使用 WSL 在 Windows 上安装 Linux](https://learn.microsoft.com/en-us/windows/wsl/install)

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

