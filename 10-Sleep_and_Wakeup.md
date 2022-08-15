# Sleep and Wake up
## 1. sleep and wake up基本概念
<font color=#0000ff>调度和锁有助于让一个进程对另一个进程的不可见，而睡眠和唤醒（sleep and wake up）机制有助于进程之间进行交互。</font>睡眠和唤醒通常被称为序列协调（sequence coordination） 或条件同步（conditional synchronization） 机制。

## 2. 睡眠唤醒机制
### 2.1 sleep和wakeup的使用逻辑
sleep的使用逻辑是：检查状态，不符合则调用sleep阻塞当前线程；

wakeup的使用逻辑是：改变状态，调用wakeup唤醒对应线程。

### 2.2 sleep channel的作用
Sleep(chan)睡眠chan上，chan是一个人为设定的值，称为等待通道(wait channel)。Sleep使调用进程进入睡眠状态，释放CPU进行其他工作。Wakeup(chan)唤醒所有在chan上sleep的线程（如果有的话），使它们的sleep调用返回。如果没有线程在chan上等待，则wakeup不做任何事情。

我们在调用wakeup的时候，需要传入与调用sleep函数相同的sleep channel。当我们调用sleep函数时，我们通过这个sleep channel表明我们等待的特定事件，所以当调用wakeup时，我们希望能通过这个channel值来表明想唤醒哪个线程。

### 2.3 lost wake问题
lost wakeup问题的引发过程：
1. 线程1首先检查状态，发现不符合；
2. 线程2改变状态，调用wakeup方法想要唤醒线程1。但是这时线程1还未调用sleep方法，其还未被阻塞，所以该wakeup无法在channel中找到线程1，wakeup丢失；
3. 最后线程1调用sleep方法，阻塞线程，且无法再被唤醒。

避免lost wakeup的方法：

采用锁来解决，用锁来保证线程1检查状态和调用sleep这一过程的原子性。也就是说，线程1和线程2在检查/改变状态之前都要先获取锁，sleep/wakeup调用后再释放锁，保证线程2无法在线程1检查状态之后到调用sleep方法之前调用wakeup。具体过程为：
1. 线程1获取锁，检查状态，发现不符合；
2. 线程2想要改变状态，但此时锁已被线程1获取，开始自旋；
3. 线程1调用sleep方法，转为SLEEPING状态，释放锁；
4. 线程2获取锁，调用wakeup唤醒线程1，将它转为RUNABLE状态，再释放锁。

## 3. 睡眠唤醒机制的应用
### 3.1 pipe中的睡眠唤醒机制
pipe中的睡眠唤醒机制体现了线程之间的交互。管道的运行过程：写入管道一端的字节被复制到内核缓冲区，然后可以从管道的另一端读取。

1. 每个管道由一个结构体 pipe表示，它包含一个锁和一个数据缓冲区。

2. 假设对piperead和pipewrite的调用同时发生在两个不同的CPU上。Pipewrite首先获取管道的锁pi->lock，它保护了计数、数据和相关的不变式。然后，Piperead 也试图获取这个锁，但是不会获取成功。它在acquire中循环，等待锁的到来。

3. 当piperead等待时，pipewrite会循环写，依次将每个字节添加到管道中。在这个循环中，可能会发生缓冲区被填满的情况。在这种情况下，pipewrite调用wakeup来提醒所有睡眠中的reader有数据在缓冲区中等待，等待reader从缓冲区中取出一些字节。此时Sleep函数内会释放pi->lock，然后pipwrite进程睡眠。

4. 现在pi->lock可用了，piperead设法获取它并进入它的临界区：它发现pi->nread != pi->nwrite，所以它进入for循环，开始read工作。现在又可写了，所以 piperead 在返回之前调用wakeup来唤醒在睡眠的writer。

（注意，等待通道如果有多个reader 和 writer，除了第一个被唤醒的进程外，其他进程都会看到条件仍然是假的，然后再次睡眠。）

### 3.2 kill
目标进程运行到内核代码中能安全停止运行的位置时，会检查自己的killed标志位，如果设置为1，目标进程会自愿的执行exit系统调用。

例如一个进程正在更新一个文件系统并创建一个文件的过程中，进程不适宜在这个时间点退出，因为我们想要完成文件系统的操作，之后进程才能退出，因此该过程中killed标志位不会被检查。

所以kill系统调用并不是真正的立即停止进程的运行，它更像是这样：如果进程在用户空间，那么下一次它执行系统调用它就会退出，又或者目标进程正在执行用户代码，当时下一次定时器中断或者其他中断触发了，进程才会退出。所以从一个进程调用kill，到另一个进程真正退出，中间可能有很明显的延时。
