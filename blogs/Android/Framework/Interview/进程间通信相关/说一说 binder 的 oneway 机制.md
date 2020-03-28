---
说一说 binder 的 oneway 机制
---

1. binder 的 oneway 是什么意思？
2. oneway 有哪些特性？
3. 它的实现原理是怎样的？

```java
interface IRemoteCaller{
	void publishBinder(ICallback callback);
}
public void publishBinder(ICallback callback){
    mRemote.transact(Stub.TRANSACTION_publishBinder, _data, _reply, 0);
}

oneway interface IRemoteCaller{
    void publishBinder(ICallback callback);
}
public void publishBinder(ICallback callback){
    mRemote.transact(Stub.TRANSACTION_publishBinder, _data, null, IBinder.FLAG_ONEWAY);
}
```

```c++
// 带 oneway
status_t transact(int32_t handle, uint32_t code, Parcel& data, Parcel* reply, uint32_t flags){
	writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    waitForResponse(NULL, NULL);
}
// 不带 oneway
status_t transact(int32_t handle, uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags){
    writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    waitForResponse(reply);
}
```

![](https://i.loli.net/2020/03/28/8ENCcGDdYVlUKQm.png)

例子：

```java
final void scheduleLauncherActivity(Intent intent, IBinder token, ...){
    mRemote.transact(SCHEDULT_LAUNCH_ACTIVITY_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);
}
oneway interface IWindow{}
oneway interface IServiceConnection{}
oneway interface IIntentReceiver{}
```

#### 总结

1. oneway 是异步 binder 调用
2. server 端串行化处理
3. oneway 的实现机制

