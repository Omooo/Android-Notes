---
Binder 对象跨进程传递的原理是怎样的？
---

1. binder 传递有哪些方式？
2. binder 在传递过程中是怎么存储的？
3. binder 对象序列化和反序列化过程？
4. binder 对象传递过程中驱动层做了什么？

先从 AIDL 入口：

```java
interface IRemoteCaller{
	void publishBinder(ICallback callback);
}
```

```java
public void publishBinder(ICallback callback){
    Parcel_data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    try{
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeStrongBinder(callback!=null?callback.asBinder():null);
        mRemote.transact(Stub.TRANSACTION_publishBinder, _data, _reply, 0);
        _reply.readException();
    }finally{
        _reply.recycle();
        _data.recycle();
    }
}
```

```java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags){
    String descriptor = DESCRIPTOR;
    switch(code){
        case TRANSACTION_publishBinder:{
            data.enforceInterface(descriptor);
            ICallback _arg0;
            _arg0 = ICallback.Stub.asInterface(data.readStrongBinder());
            this.publishBinder(_arg0);
            reply.writeNoException();
            return true;
        }
    }
}
```

```java
public final void writeStrongBinder(IBinder val){
	nativeWriteStrongBinder(mNativePtr, val);
}
void Parcel writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object){
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if(parcel!=NULL){
        const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
        if(err!=NO_ERROR){
            signalExceptionForError(env, clazz, err);
        }
    }
}
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj){
    if(env->IsInterfaceOf(obj, gBinderOffsets.mClass)){
        env->GetLongField(obj, gBinderOffsets.mObject);
        return jbh!=NULL?jbh->get(env, obj):NULL;
    }
    if(env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)){
        return (IBinder*)
            env->GetLongField(obj, gBinderProxyOffsets.mObject);
    }
    return NULL;
}
```

```c++
status_t Parcel:writeStrongBinder(const sp<IBinder>&val){
	return flatten_binder(ProcessState::self(), val, this);
}
status_t flatten_binder(sp<ProcessState>&proc, sp<IBinder>&binder, Parcel* out){
    flat_binder_object obj;
    IBinder *local = binder->localBinder();
    obj.type = BINDER_TYPE_BINDER;
    obj.cookie = reinterpret_cast<uintptr_t>(local);
    return finish_flatten_binder(binder, obj, out);
}
status_t finish_flatten_binder(sp<IBinder>& binder, flat_binder_object& flat, Parcel* out){
    return out->writeObject(flat, false);
}
status_t Parcel::writeObject(const flat_binder_object& val, bool nullMetaData){
    *reinterpret_cast<flat_binder_object*>(mData+mDataPos)=val;
    mObjectsSize++;
    return finishWrite(sizeof(flat_binder_object));
}
status_t Parcel::finishWrite(size_t len){
    mDataPos += len;
    return NO_ERROR;
}
```

Parce 到了驱动层是如何处理的？

```c++
static void binder_transaction(struct binder_proc *proc, ...){
	for(;offp<off_end;offp++){
        fp=(struct flat_binder_object *)(t->buffer->data+*offp);
        switch(fp->type){
            case BINDER_TYPE_BINDER:{
                struct binder_node *node = binder_get_node(proc, fp->binder);
                if(node==NULL){
                    node=binder_new_node(proc,fp->binder,fp->cookie);
                }
                ref = binder_get_ref_for_node(target_proc,node);
                if(fp->type==BINDER_TYPE_BINDER)
                    fp->type=BINDER_TYPE_HANDLE;
                fp->handle=ref->desc;
            }break;
        }
    }
}
```

读：

```java
public final IBinder readStrongBinder(){
	return nativeReadStrongBinder(mNativePtr);
}
```

```c++
jobject Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr){
	Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
	if(parcel!=NULL){
		return javaObjectForIBinder(env, parcel->readStrongBinder());
	}
	return NULL;
}
sp<IBinder> Parcel::readStrongBinder() const{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}
status_t unflatten_binder(sp<ProcessState>& proc, Parcel& in,sp<IBinder>* out){
    const flat_binder_object* flat = in.readObject(false);
    switch(flat->type){
        case BINDER_TYPE_BINDER:
        	*out = reinterpret_cast<IBinder*>(flat->cookie);
            return ...;
        case BINDER_TYPE_HANDLE:
            *out = proc->getStrongProxyForHandle(flat->handle);
            return ...;
    }
}
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle){
    sp<IBinder> result;
    handle_entry* e = lookupHandleLocked(handle);
    IBinder* b = e->binder;
    if(b == NULL){
        b = new BpBinder(handle);
        e->binder = b;
        result = b;
    }
    return result;
}
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val){
    if(val->checkSubclass(&gBinderOffsets)){
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        return object;
    }
    object = env->NewObject(gBinderProxyOffsets.mClass, ...);
    env->SetLongField(object, gBinderProxyOffsets.mObject, (jlong)val.get());
    return object;
}
```

#### binder 是怎么跨进程传输的？

1. Parcel 的 writeStrongBinder 和 readStrongBinder
2. binder 在 Parcel 中存储原理，flat_binder_object
3. 说清楚 binder_node，binder_ref
4. 目标进程根据 binder_ref 的 handle 创建 BpBinder
5. 由 BpBinder 再往上到 BinderProxy 到业务层的 Proxy