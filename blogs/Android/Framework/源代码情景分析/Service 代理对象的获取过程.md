---
Service 代理对象的获取过程
---

Service 组件将自己注册到 ServiceManager 中之后，它就在 Service 进程中等待 Client 进程将进程间通信请求发送过来。Client 进程为了和运行中 Service 进程中的 Service 组件通信，首先要获得它的一个代理对象，这是通过 ServiceManager 提供的 Service 组件查询服务来实现的。

ServiceManager 代理对象的成员函数 getService 提供了获取一个 Service 组件的代理对象的功能，而 ServiceManager 代理对象可以通过 Binder 库提供的函数 defaultServiceManager 来获得。在调用 ServiceManager 代理对象的成员函数 getService 来获得一个 Service 组件的代理对象时，需要指定这个 Service 组件注册到 ServiceManager 中的名称。

ServiceManager 代理对象的成员函数 getService 的实现如下所示：

```c++
class BpServiceManager : public BpInterface<IServiceManager>
{
	virtual sp<IBinder> getService(const String16& name) const
	{
		unsigned n;
		for (n = 0; n < 5; n++) {
			sp<IBinder> svc = checkService(name);
			if (svc != NULL) return svc;
			sleep(1);
		}
		return NULL;
	}
}
```

这个函数最多会尝试五次来获得一个名称为 name 的 Service 组件的代理对象。如果上一次获取失败，就会 sleep 1 秒后重新获取。

ServiceManager 代理对象的成员函数 checkService 的实现如下所示：

```c++
class BpServiceManager : public BpInterface<IServiceManager>
{
	virtual sp<IBinder> checkService(const String16& name) const
	{
		Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
		return reply.readStrongBinder();
	}
}
```

ServiceManager 是统一在函数 svcmgr_handler 中处理来自 Client 进程的进程间通信请求的，它处理操作代码为 CHECK_SERVICE_TRANSACTION 的进程间通信请求的过程如下所示：

```c++
int svcmgr_handler(struct binder_state *bs, ...) {
    struct svcinfo *si;
    switch(txn->code) {
        case SVC_MGR_GET_SERVICE:
        case SVC_MGR_CHECK_SERVICE:
            ptr = do_find_service(bs, ...);
    }
}

void *do_find_service(struct binder_state *bs, ...)
{
    struct svcinfo *si;
    si = find_svc(s, len);
    if (si && si->ptr) {
        return si->ptr;
    } else {
        return 0;
    }
}
```

调用函数 find_svc 来查找与字符串 s 对应的一个 svcinfo 结构体 si，它通过遍历已注册 Service 组件列表 svclist 来查找与字符串 s 对应的一个 svcinfo 结构体，然后返回它的成员变量 ptr。

结构体 svcinfo 的成员变量 ptr 保存的是一个引用了注册到 ServiceManager 中的 Service 组件的 Binder 引用对象的句柄值。当 ServiceManager 将这个句柄值返回给 Binder 驱动程序时，Binder 驱动程序就可以根据它找到相应的 Binder 引用对象，接着找到该 Binder 引用对象所引用的 Binder 实体对象，最后 Binder 驱动程序就可以在请求获取该 Service 组件的代理对象的 Client 进程中创建另一个 Binder 引用对象了。