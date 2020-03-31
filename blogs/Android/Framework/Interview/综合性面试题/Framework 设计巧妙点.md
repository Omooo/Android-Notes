---
Framework 设计巧妙点
---

1. 对 Framework 的涉猎有一定的广度和深度
2. 对系统设计有自己的总结和思考

#### 例子

1. Binder 调用，模糊进程边界
2. Bitmap 大图传输，高性能
3. Zygote 创建进程，资源共享
4. Intent 解耦，模糊进程

Binder 调用：

1. 请求转发
2. Binder 对象的传递

Bitmap 大图传输：

```c++
static jboolean Bitmap_writeToParcel(JNIEnv* env, jobject, ...){
    int fd = androidBitmap->getAshmemFd();
    if(fd>=0&&!isMutable&&p->allowFds()){
        p->writeDupImmutableBlobFileDescriptor(fd);
        return JNI_TRUE;
    }
}
static jobject Bitmap_createFromParcel(JNIEnv* env, jobject parcel){
    if(blob.fd()>=0 && (blob.isMutable()||!isMutable) && (size>=ASHMEM_BITMAP_MIN_SIZE)){
        nativeBitmap = GraphicsJNI::mapAshmemPixelRef(env, bitmap.get(), ctable, dupFd, const_cast<void*>(blob.data()), !isMutable);
    }
}
```

Zygote 创建进程：

```c++
int main(int argc, char* const argv[]){
	runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
}
// ZygoteInit#main
public static void main(String argv[]){
    preload();
    startSystemServer(abList, socketName);
    runSelectLoop(abiList);
}
```

Intent 解耦：

```java
Intent registerReceiver(..., IIntentReceiver receiver, IntentFilter filter, ...){
	ReceiverList rl = mRegistedReceivers.get(receiver.asBinder());
	if(rl == null){
		rl = new ReceiverList(this, ..., yserId, receiver);
		rl.app.receivers.add(rl);
		mRegisteredReceivers.put(receiver.asBinder(), rl);
	}
	BroadcastFilter bf = new BroadcastFilter(filter, rl, callerOackage, ...);
	rl.add(bf);
	mReceiverResolver.addFilter(bf);
}
```

```java
int broadcastIntentLocked(ProcessRecord callerApp, ...){
	List receivers = null;
    List<BroadcastFilter> registeredReceivers = null;
    registeredReceivers = mReceiverResolver.queryIntent(intent, ...);
    
    int NR = registeredReceivers.size();
    final BroadcastQueue queue = broadcastQueueForIntent(intent);
    BroadcastRecord r = new BroadcastRecord(queue, intent, ...);
    queue.enqueueParallelBroadcastLocked(r);
    queue.scheduleBroadcastLocked();
    return ActivityManager.BROADCAST_SUCCESS;
}
```

```java
while(mParallelBroadcasts.size()>0){
	r = mParallelBroadcasts.remove(0);
    final int N = r.receivers.size();
    for(int i=0;i<N;i++){
        Object target = r.receivers.get(i);
        deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
    }
}
```

