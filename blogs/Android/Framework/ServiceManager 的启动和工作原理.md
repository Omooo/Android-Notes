---
ServiceManager 的启动和工作原理
---

1. ServiceManager 启动流程是怎样的？
2. 怎么获取 ServiceManager 的 binder 对象？
3. 怎么向 ServiceManager 添加服务？
4. 怎么从 ServiceManager 获取服务？

#### ServiceManager 的启动

1. 启动进程
2. 启用 binder 机制
3. 发布自己的服务
4. 等待并响应请求

##### 启动进程

```ini
service servicemanager /system/bin/servicemanager
clas core
user system
group system
critical
//...
```

```c
int main(int argc, char **argv) {
	struct binder_status *bs;
    // 1.打开 binder 驱动
	bs = binder_open(128*1024);
    // 2.把自己注册为上下文管理者
	binder_become_context_manager(bs);
    // 3.进入 loop 循环，不断等待请求、处理请求
	binder_loop(bs, svcmgr_handler);
	return 0;
}
```

```c
struct binder_status *binder_open(size_t, mapsize) {
	struct binder_state *bs;
	bs = malloc(sizeof(*bs));
    // 1.打开 binder 驱动，返回一个 fd
	bs->fd = open("/dev/binder", O_RDWR);
	bs->mapsize = mapsize;
    // 2.给 fd 映射一块内存，128 kb
	bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
	return bs;
}
```

```c
// 注册成上下文，其实就是告诉 binder 驱动，ServiceManager 已经就绪了
int binder_become_context_manager(struct binder_state *bs) {
	return ioctl(bs->fd, BINDER_SET_CONTEXT_MSR, 0);
}
```

binder_loop 分为两个阶段，第一阶段主要是注册为 binder 线程：

```c
void binder_loop(struct binder_state *bs, binder_handler func) {
	uint32_t readbuf[32];
    // 把当前线程注册为 binder 线程，也就是告诉 binder 驱动，当前线程可以处理 binder 请求的
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));
    //...
}
```

```c
int binder_write(struct binder_state *bs, void *data, size_t len) {
    // BINDER_WRITE_READ 表示可读可写
    // write_size > 0 执行写
    // read_size > 0 执行读
    // write_size > 0 && read_size > 0，先写再读
	struct binder_write_read bwr;
	bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    return res;
}
```

第二阶段，就是读请求处理请求了：

```c
void binder_loop(struct binder_state *bs, binder_handler func) {
    struct binder_write_read bwr;
    //...
    bwr.write_size = 0;
    for(;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_buffer = (unitptr_t) readbuf;
        ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        // 解析请求，并在回掉函数 func 处理请求
        binder_parse(bs, 0, (uintptr_t)readbuf, bwr.read_consumed, func);
    }
}
```

#### 如何获取 ServiceManager？

以 SurfaceFlinger 为例：

```c
sp<IServiceManager> defaultServiceManager() {
	if(gDefaultServiceManager != null) {
		return gDefaultServiceManager;
	}
	while(gDefaultServiceManager == null) {
		gDefaultServiceManager = getContextObject();
		if(gDefaultServiceManager==null) sleep(1);
	}
}
```

```c
sp<IBinder> ProcessState::getContextObject(...) {
    // 返回 ServiceManager 的 Proxy 对象，注意 handler 值为 0
	return getStrongProxyForHandle(0);
}
```

```c
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle) {
	// 以 binder handle 值作为索引，获取数组里面对应的 handle_entry
    handle_entry* e = loopupHandleLocked(handle);
	IBinder* b = e->binder;
	if(b == NULL) {
        // ServiceManager 的 Proxy 对象就是 BpBinder(0);
		b = new BpBinder(handle);
		e->binder = b;
	}
	return b;
}
```

#### 怎么向 ServiceManager 添加服务？

```c
// 添加服务，Service 名称和 Service Binder 对象
status_t addService(const String16& name, const sp<IBinder>& service, ...) {
	//...
    // 通过 remote 函数获取 BpBinder 对象，然后调用 BpBinder 的 transact 函数
	remote() -> transact(ADD_SERVICE_TRANSACTION, data, &reply);
}
```

```c
status_t BpBinder::transact(unit32_t code, const Parcel& data, Parcel* reply, ...) {
    // IPCThreadState 线程单例，直接跟 binder 驱动交互
	IPCThreadState::self()->transact(mHandle, code, data, ...);
}
```

在 ServiceManager 收到请求之后，如何处理？

```c
int svcmgr_handler(..., struct binder_transaction_data *txn, ...) {
	switch(txn->code) {
		//...
		case SVC_MSR_ADD_SERVICE:
            // 注册到 ServiceManage 的虽然传递的是 binder 实体对象
            // 但是真正 ServiceManager 收到的只是一个 handle 值
            // ServiceManager 会根据 handle 值还有 binder 相关信息封装成一个数据结构插入到单链表里
            do_add_service(bs, s, len, handle, ...);
            break;
	}
}
```

#### 怎么从 ServiceManager 获取服务？

关于怎么获取服务呢，和注册服务差不多，都需要向 ServiceManager 发起一个 binder 调用。

```java
public static IBinder getService(String name) {
	IBinder service = sCache.get(name);
	if(service != null) {
		return service;
	}else {
        // 获取 ServiceManager 的 binder 对象，然后调用它的 getService 函数
		return getIServiceManager().getService(name);
	}
	return null;
}
```

ServiceManager 在收到请求之后，是如何处理的呢？

```c
int svcmgr_handler(..., struct binder_transaction_data *txn, ...) {
	uint32_t handle;
	switch(txn->code) {
		case SVC_MGR_GET_SERVICE:
            // 拿到服务的名称
			s = bio_get_string16(msg, &len);
            // 找到对应服务的 handle 值
			handle = do_find_service(bs, s, len, ...);
            // 返回 handle 值，Client 端再根据 handle 值封装一个 BpBinder 就行了
			bio_put_ref(reply, handle);
			return 0;
	}
}
```

