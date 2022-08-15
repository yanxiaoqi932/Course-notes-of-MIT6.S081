# OS Organization
## 1. OS的要求：复用的同时，具有强隔离性

## 2. 操作系统对硬件进行了抽象
### 2.1 进程是CPU的抽象
每个cpu核一次只能运行一个进程，如果想要运行多个进程的话，那么就需要时间复用，也就是每隔一段时间换一个进程运行。

为了帮助强制隔离，进程抽象给程序提供了一种认为它独占机器的错觉，进程为程序提供了专用内存系统或地址空间，其他进程无法读写。

Xv6使用页表（由硬件实现）给每个进程提供自己的地址空间。RISC-V页表将虚拟地址（RISC-V指令操作的地址）翻译（或“映射”）为物理地址（CPU芯片发送到主存储器的地址）。

进程下有线程来执行它的指令。

每个进程都有两个堆栈：一个用户堆栈和一个内核堆栈(p->kstack)，进程处于不同状态时就使用对应的堆栈。

<font color=#FF0000>补充：[CSDN 进程、线程与CPU的关系](https://blog.csdn.net/nandao158/article/details/105896980)

一个应用程序可能有一个或多个进程；进程是运行的基本单元，其中资源包括:CPU、内存空间、 磁盘 IO等；进程内部含有多个线程，线程共享进程的所有资源；一个CPU核一次只能执行一个线程，它通过并行处理（例如时间片轮转）的方法来并行执行多个线程。

另外，每个用户进程在内核中都有一个对应的内核线程。
</font>
### 2.2 exec是内存的抽象
OS提供内存隔离，因此控制了应用程序与物理硬件之间的交互，因此应用程序需要调用exec来间接访问内存。

关于exec指令：exec不像fork那样建立一个与调用进程并发的新进程（子进程），而是而是用新进程取代原来的进程
### 2.3 文件是对磁盘块的抽象
应用程序不直接对磁盘进行读写，而是通过文件来访问磁盘，文件具体映射到磁盘的哪一块，由操作系统决定，一个磁盘块只能出现在一个文件中。文件抽象接口提供了不同用户以及同一用户的不同进程之间的强隔离。

## 3. OS应该是防御性的
### 3.1 应用程序不能破坏操作系统
### 3.2 应用程序不能打破隔离
### 3.3 因此操作系统与应用程序之间必须强隔离
硬件支持：1. 用户（内核）模式； 2. 虚拟内存

## 4. 内核态/用户态
内核态可以执行特权指令（直接操作硬件的指令，比如配置页表寄存器，禁止时钟中断等），用户态不行。
### 4.1 内存
每个进程有自己的内存布局，相互独立，有很强的内存隔离。

每个进程有自己的虚拟内存，OS将它们各自的虚拟内存地址映射到物理内存地址，它们并不能知道彼此真实的物理地址，使得它们之间不能互相访问彼此的内存。

### 4.2 控制权的转移
用户程序将控制权转移到内核中来执行内核操作。例如用户使用fork操作，但用户侧实际上并没有fork这一操作，OS通过RISC-V中一个名为ecall的指令，进入内核完成第一次内核转换；进入内核后，调用内核中的syscall指令，调用fork。

系统调用过程如下图所示：
![xv6系统调用过程](./images/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B.png)

### 4.3 内核的安全性
内核作为操作系统的核心部分，它也被称为可信任计算基础，它必须正确，必须保证自己没有bug，因为攻击者会利用这些bug来攻击系统甚至控制内核。因此内核必须将用户程序当成是恶意的。

应用程序不可能直接操作硬件，而是必须通过内核来间接使用硬件资源。例如：<font color=#FF0000>一个进程可以通过执行RISC-V的ecall指令来进行系统调用，此指令提高了硬件特权级别，并将程序计数器更改为一个由内核定义的入口点。入口点上的代码切换到一个内核堆栈，并执行实现系统调用的内核指令。当系统调用完成时，内核切换回用户堆栈，并通过调用sret指令返回到用户空间，这降低了硬件特权级别，并在系统调用指令之后恢复执行用户指令。</font>

### 4.4 宏内核与微内核
宏内核的内核范围较大，除了时钟、中断、cpu切换等核心功能之外，内核还包括进程管理、存储器管理和设备管理等部分，它的优点是许多子模块可以紧密结合，性能更高；缺点是内核范围过大，故而存在的bug会更多。常用的windows和Linux，以及许多服务器的操作系统内核都属于宏内核。

微内核的内核范围较小，它一般只包括时钟、中断、cpu切换等核心功能。它的优点是内核小，bug较少，在小设备中累赘少；缺点是需要频繁的在用户态和内核态之间交换，从而降低了性能。微内核常常应用于嵌入式设备中。

## 5. trace实验
### 5.1 C语言的左移和右移
[CSDN C语言左移和右移](https://blog.csdn.net/shimmy_lee/article/details/84429276?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-84429276-blog-85624722.pc_relevant_sortByStrongTime&spm=1001.2101.3001.4242.1&utm_relevant_index=2)

[CSDN C语言的负数表示方法：补码](https://blog.csdn.net/Piconjo/article/details/106763643)

一般逻辑左移，算术右移；左移乘2，右移除2：
![逻辑移位和算术移位](./images/%E5%B7%A6%E7%A7%BB%E5%8F%B3%E7%A7%BB.png)

### 5.2 根据系统调用过程实现trace
[CSDN trace的实现](https://blog.csdn.net/qq_33095733/article/details/123608334)

根据系统调用过程来在内核中实现一个trace，需要对user.h, usys.S, syscall.c, syscall.h, sysproc.c, proc.c几个文件中加入trace。

