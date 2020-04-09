---
init 进程
---

#### 前言

所有的 Unix 系统都有一个特殊的进程，它是内核启动完毕后，用户态中启动的第一个进程，它负责启动系统，并作为系统中其他进程的祖先进程。

不过 Android 中的 init 和 Linux 中的 init 还是有很大不同的，其中最重要的不同之处在于：Android 中的 init 支持系统属性，并使用一些指定的 rc 文件来规范它的行为。

另外，init 还扮演着其他的角色 --- 它要化身为 ueventd 和 watchdogd。这两个重要的核心服务也是由 init 这个二进制可执行文件来实现的 --- 通过符号链接的方式加载。

#### 系统属性

Android 的系统属性提供了一个可全局访问的配置设置仓库，这就是一个类似 windows 注册表的东西，每个属性实际上也就是一个 key-value 对。

属性文件的只读文件描述符被设为可以被子进程继承，这使得系统中任何一个进程都能方便的访问到系统属性，尽管只能读。

为了能提供写属性服务请求的服务，init 专门打开了一个专用的 Unix domain socket --- /dev/socket/property_service。只要能连上这个 socket，任何人都可以对它进行写操作，不过写操作是转发给 init 让 init 去执行的。

#### 启动服务

init fork 出子进程后，会设置服务子进程的权限，并设置该服务子进程用来获取输入的 socket 及配置环境变量、I/O 优先级。只有当所有这些操作执行完毕之后，init 才会去执行服务本身的二进制可执行文件。

在服务启动之后，init 会维持一个指向该服务的父链接。这样，一旦该服务停止运行或者崩溃了，init 就会收到一个 SIGCHLD 信号，并注意到这一事件，然后重启该服务。onrestart 关键字会使 init 在各个指定的服务之间建立连接，当特定的服务需要重启时，init 会去运行这个使用了 onrestart 关键字的 service 语句块中的命令，或者重启与该服务有依赖关系的服务。

#### Init 的执行流程

初始化流程：

1. 检查自身这个二进制可执行文件是不是被当成 ueventd 或在 watchdogd 调用的。如果是的话，余下的执行流程就会转到相应的守护进程的主循环那里去
2. 创建 /dev、/proc 和 /sys 等目录，并且 mount 它们
3. 调用 property_init() 函数
4. 调用 init_parse_config_file() 函数去解析 /init.rc 脚本文件
5. init 会把 init.rc 文件中各个 on 语句块里规定的 action 以及内置的 action 添加到一个名为 action_queue 的队列里去

最后，主循环中将会逐个执行 init.rc 中的所有命令，然后，init 将在它的生命周期中的大多数时间里处于休眠状态，只有在必要的时候 init 进程才会被唤醒。

主循环：

init 的主循环相当简洁，一共由三个步骤组成：

1. execute_one_command() 从队列 action_queue 的头部取出一个 action（如果有的话），并执行之

2. restart_processes() 逐个检查所有已经注册过的服务，并在必要时重启

3. 安装并轮询监听如下三个 socket 描述符：

   - property_set_fd（/dev/socket/property_service）

     这个 socket 是用来让想要设置某个属性的客户端进程，通过它把要设置的 key-value 发送给 init。

   - keychord_fd（/dev/keychord，如果存在的话）

     处理启动服务的组合键的。

   - signal_recv_fd

     它是一对 socketpair 中的一端，创建它是为了处理因子进程死亡而发来的 SIGCHLD 信号。当 init 收到这个信号之后，就会释放该进程残留的资源，防止它成为僵尸进程。

必须要强调的一个重点是：除了它监听的这三个文件描述符之外，/init 是不会从其他地方接受输入的。换而言之，除此之外再也没有其他方法可以影响 /init 的操作行为了。

#### init 的其他角色

init 还承担了其他两个角色，那就是 ueventd 和 watchdogd。虽然完成这些任务所使用的二进制可执行文件和 /init 进程的二进制可执行文件是同一个文件，但是代码的执行路径确实完全不同的。

ueventd:

在作为 ueventd 运行时，init 这个二进制可执行文件是用来管理硬件设备的，它会使用另一些初始化脚本 /ueventd.rc，这些配置文件只记录了相关文件的路径及其应被设置为什么样的权限。

watchdogd:

它被用作硬件 watchdog 定时器在用户态中的接口。设置一个超时变量，每隔一段时间就发送一个 keepalive 信号。如果超过了超时变量规定的时间，watchdogd 还没能及时发送 keepalive 信号，硬件 watchdog 定时器就会发出一个中断，要求内核重启。尽管有那么点极端的感觉，但是我们要知道唯一能让 watchdog 不能及时发送 keepalive 信号的原因只有系统已经挂了。

