---
Looper 的副业
---

```c++
int addFd(int fd, int ident, int events, sp<LooperCallback>& callback, ...){
	epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, &eventItem);
}
void NativeMessageQueue::setFileDescriptorEvents(int fd, int events){
    mLooper->addFd(fd, Looper::POLL_CALLBACK, looperEvents, this, reinterpret_cast<void*>(events));
}
void addOnFileDescriptorEventListener(FileDescriptor fd, int events, OnFileDescriptorEventListener listener){...}
```

Framework 里面有哪些地方用到了副业？

```c++
status_t NativeDisplayEventReceiver::initialize(){
	mMessageQueue->getLooper()->addFd(mReceiver.getFd(), 0, EVENT_INPUT, this, NULL);
}
```

#### 总结

1. Looper 里可以监听其他描述符
2. 创建管道，跨进程传数据，用 Looper 监听描述符事件