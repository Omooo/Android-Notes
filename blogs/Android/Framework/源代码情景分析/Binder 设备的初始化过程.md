---
Binder 设备文件
---

#### 初始化过程

Binder 设备的初始化过程是在 Binder 驱动程序的初始化函数 binder_init 中进行的。

它首先在目标设备上创建了一个 /proc/binder/proc 目录，每一个使用了 Binder 进程间通信机制的进程在该目录下都对应有一个文件，这些文件是以进程 ID 来命名的，通过它们就可以读取到各个进程的 Binder 线程池、Binder 实体对象、Binder 引用对象以及内核缓冲区等信息。然后调用函数 misc_register 来创建一个 Binder 设备，最后 Binder 驱动程序在目标设备上创建了一个 Binder 设备文件 /dev/binder，这个设备文件的操作方法列表是由全局变量 binder_fops 指定的。全局变量 binder_fops 为 Binder 设备文件 /dev/binder 指定文件打开、内存映射和 IO 控制函数分别是 binder_open、binder_mmap 和 binder_ioctl。

#### 打开过程

一个进程在使用 Binder 进程间通信机制之前，首先要调用函数 open 打开设备文件 /dev/binder 来获得一个文件描述符，然后才能通过这个文件描述符来和 Binder 驱动程序交互，继而和其他进程执行 Binder 进程间通信。

当进程调用函数 open 打开设备文件 /dev/binder 时，Binder 驱动程序中的函数 binder_open 就会被调用，它会为进程创建一个 binder_proc 结构体 proc，并把它加入到一个全局 hash 队列 binder_procs 中。Binder 驱动程序将所有打开了设备文件 dev/binder 的进程都加入到全局 hash 队列 binder_procs 中。最后会在目标设备上的 /proc/binder/proc 目录下创建一个以进程 ID 为名称的只读文件，通过读取这个文件就可以获得进程 PID 的 Binder 线程池、Binder 实体对象、Binder 引用对象以及内核缓冲区等信息。

#### 内存映射过程

进程打开了设备文件 /dev/binder 之后，还必须要调用函数 mmap 把这个设备文件映射到进程的地址空间，然后才可以使用 Binder 进程间通信机制。设备文件 /dev/binder 对应的是一个虚拟设备，将它映射到进程的地址空间的目的并不是对它的内容感兴趣，而是为了为进程分配内核缓冲区，以便它可以用来传输进程间通信数据。

当进程调用函数 mmap 将设备文件 /dev/binder 映射到自己的地址空间时，Binder 驱动程序中的函数 binder_mmap 就会被调用。