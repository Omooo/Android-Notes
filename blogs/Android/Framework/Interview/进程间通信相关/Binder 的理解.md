---
谈谈你对 Binder 的理解？
---

1. binder 是干嘛的？
2. binder 存在的意义是什么？
3. binder 的架构原理是怎样的？

#### Binder 是干嘛的？

进程间通信。

#### Binder 存在的意义是什么？

1. 性能
2. 方便易用
3. 安全

#### 架构原理

通信架构：

![](https://i.loli.net/2020/03/27/REqCWzQSnokHKFw.png)

#### 进程如何启用 Binder 机制？

1. 打开 binder 驱动
2. 内存映射，分配缓冲区
3. 启动 binder 线程

```c++
// ServiceManager#main
int main(int argc, char **argv){
    struct binder_state *bs;
    bs = binder_open(128*1024);
    binder_become_context_manager(bs);
    binder_loop(bs, svcmgr_handler);
    return 0;
}
void binder_loop(struct binder_state *bs, binder_handler func){
    // 把当前线程注册为 binder 线程，也就是 SM 的主线程
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));
    for(;;){
        bwr.read_size = sizeof(readbuf);
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
    }
}
```

以 SurfaceFlinger 注册服务为例：

```c++
int main(int, char**){
    // 启动 binder 机制
	sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();
    // 初始化服务
    sp<SurfaceFlinger> flinger = new SurfaceFlinger();
    flinger->init();
    // 注册服务
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName(), flinger, false));
    flinger->run();
    return 0;
}
sp<IServiceManager> defaultServiceManager(){
    while(gDefaultServiceManager==NULL){
        ProcessState::self()->getContextObject(NULL);
        if(gDefaultServiceManager==NULL){
            sleep(1);
        }
    }
    return gDefaultServiceManager;
}
status_t addService(const String16& name, const sp<IBinder>& service, ...){
    Parcel data, reply;
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptior());
    data.writeString16(name);
    data.writeStrongBinder(service);
    data.writeInt32(allowlsolated?1:0);
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR?reply.readExceptionCode():err;
}
status_t IPCThreadState::transact(int32_t handle, unit32_t code, ...){
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, ...);
    if((flag&TF_ONE_WAY)==0){
        if(reply){
            err=waitForResponse(reply);
        }else{
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    }else{
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
status_t BnServiceManager::onTransact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags){
    switch(code){
        case ADD_SERVICE_TRANSACTION:{
            CHECK_INTERFACE(IServiceManager, data, reply);
            String16 which = data.readString16();
            sp<IBinder> b = data.readStrongBinder();
            status_t err = addService(which, b);
            reply->writeInt32(err);
            return NO_ERROR;
        }break;
    }
}
```

![](https://i.loli.net/2020/03/28/1qUCWh5B7vSVzAe.png)

