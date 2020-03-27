---
Android Framework 里面用到了哪些 IPC 方式？
---

1. 是否了解 Linux 常用的跨进程通信方式
2. 是否研究过 Android Framework 并了解一些实现原理
3. 是否了解 Framework 各组件之间的通信原理

#### Linux IPC 方式

1. 管道
2. Socket
3. 共享内存
4. 信号

##### 管道

1. 半双工，单向的
2. 一般是在父子进程之间使用

![](https://i.loli.net/2020/03/27/UcCtmGaYMWRoseb.png)

Framework 哪用到了管道？

Looper 里面用到了管道：

```c++
Looper::Looper(bool allowNonCallbacks){
    int wakeFds[2];
    // 创建一个管道（4.4）
    int result = pipe(wakeFds);
    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    struct epoll_event eventItem;
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
}
```

epoll 是如何监听读端事件的？

```c++
int Looper::pollInner(int timeoutMillis){
	struct epoll_event eventItems[EPOLL_MAX_EVENTS];
	int eventCount = epoll_wait(mEpollFd, eventItems, ...);
	for(int i=0;i<eventCount;i++){
		int fd = eventItems[i].data.fd;
		uint32_t epollEvents = eventItems[i].events;
		if(fd == mWakeReadPipeFd){
			if(epollEvents&EPOLLIN){
				awoken();
			}
		}
	}
	return result;
}
```

管道在哪写的呢？

```c++
void Looper::wake(){
	nWrite = write(mWakeWritePipeFd, "W", 1);
}
```

##### Socket 通信

1. 全双工的，可读可写
2. 两个进程之间无需存在亲缘关系

```java
public static void main(String argv[]){
	registerZygoteSocket(socketName);
    runSelectLoop(abiList);
}
void runSelectLoop(String abiList){
    while(true){
        Os.poll(pollFds, -1);
        for(int i=pollFds.length-1;i>=0;--i){
            if(i==0){
                // 处理新过来的连接
            }else{
                // 处理发过来的数据
                peers.get(i).runOnce();
            }
        }
    }
}
```

##### 共享内存

1. 很快，不需要多次拷贝
2. 进程之间无需存在亲缘关系

涉及进程之间大数据量传输主要就是图像相关的了：

```java
// 匿名共享内存
public MemoryFile(String name, int length){
    mLength = length;
    // ashmem_create_region(namestr, length);
    // 创建一块匿名共享内存，返回一个 fd
    mFD = native_open(name, length);
    // mmap(NULL, length, prot, MAP_SHARED, fd, 0);
    // 给这个 fd 映射到当前进程内存空间
    mAddress = native_mmap(mFD, length, PROT_READ|PROT_WRITE);
}
```

MemoryFile 的读和写：

```java
// 把共享内存的数据读到应用进程的 buffer 里面
jint android_os_MemoryFile_read(JNIEnv* env, jobject clazz, ...){
	env->SetByteArrayRegion(buffer, destOffset, count, ...);
    return count;
}
// 把应用层的 buffer 拷贝到共享内存里面
jint android_os_MemoryFile_write(JNIEnv* env, jobject clazz, ...){
	env->GetByteArrayRegion(buffer, srcOffset, count, ...);
    return count;
}
```

##### 信号

1. 单向的，发出去之后怎么处理是别人的事
2. 只能带个信号，不能带别的参数
3. 知道进程 pid 就能发信号了，也可以一次给一群进程发信号

杀掉应用进程，需要用到信号：

```java
public class Process{
	public static final void killProcess(int pid){
		sendSignal(pid, SIGNAL_KILL);
	}
}
static void SetSigChldHandler(){
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.as_handler = SigChldHandler;
    sigaction(SIGCHLD, &sa, NULL);
}
```