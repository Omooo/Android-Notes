---
应用是怎么启用 binder 机制的？
---

1. 了解 binder 是用来干嘛的？
2. 应用里面哪些地方用到了 binder 机制？
3. 应用的大致启动流程是怎样的？
4. 一个进程是怎么启用 binder 机制的？

#### 什么时候支持 binder 机制的？

#### binder 启动时机

```java
// handleChildProc(...)
static void zygoteInit() {
	commonInit();
	nativeZygoteInit();
	applicationInit(...);
}
```

```c
void nativeZygoteInit(...) {
	gCurRuntime->onZygoteInit();
}
```

```c
virtual void onZygoteInit() {
	sp<ProcessState> proc = ProcessState::self();
	proc->startThreadPool();
}
```

```c
ProcessState::ProcessState()
    // open_driver() 打开 "/dev/binder" 驱动
	:mDriverFD(open_driver()), ... {
		if(mDriverFD>=0) {
			mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, ..., mDriverFD, 0);
		}
	}
```

#### 怎么启用 Binder 机制？

1. 打开 binder 驱动
2. 映射内存，分配缓冲区
3. 注册 binder 线程
4. 进入 binder loop

