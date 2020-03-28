---
一次完整的 IPC 通信流程
---

1. 了解 binder 的整体架构原理
2. 了解应用和 binder 驱动的交互方式
3. 了解 IPC 过程中的通信协议

```c++
private static class Proxy implements inuker.com.library.IRemoteCaller{
    private android.os.IBinder mRemote;
    public void call(int code) throws RemoteException{
        mRemote.transact(Stub.TRANSACTION_call, _data, _reply, 0);
    }
}

final class BinderProxy implements IBinder{
    public boolean transact(int code, Parcel data, Parcel reply, int flags){
        return transactNative(code, data, reply, flags);
    }
}
jboolean android_os_BinderProxy_transact(INIEnv* env, jobject obj, jint code, jobject dataObj, jobject replyObj, jint flags){
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);
    IBinder* target = (IBinder)env->GetLongField(obj, gBinderProxyOffsets.mObject);
    status_t err = target->transact(code, *data, reply, flags);
}

status_t BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags){
    status_t status = IPCThreadState::self()->transact(mHandle, code, data, reply, flags);
    return status;
}
status_t IPCThreadState::transact(int32_t handle, ...){
    writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    if((flags&TF_ONE_WAY)==0){
        if(reply){
            err = waitForResponse(reply);
        }else{
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    }else{
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
status_t IPCThreadState::writeTransactionData(int32_t cmd, ...){
    binder_transaction_data tr;
    tr.target.ptr = 0;
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.data.size = data.ipcDataSize();
    tr.data.ptr.buffer = data.ipcData();
    tr.data.ptr.offsets = data.ipcObjects();
    
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));
    return NO_ERROR;
}
status_t IPCThreadState::waitForResponse(Parcel *reply, ...){
    while(1){
        talkWithDiver();
        cmd = (uint32_t)mln.readInt32();
        switch(cmd){
            case BR_TRANSACTION_COMPLETE:
                break;
            case BR_REPLY:{
                binder_transaction_data tr;
                mIn.read(&tr, sizeof(tr));
                reply->ipcSetDataReference(tr.data.ptr.buffer, tr.data_size, ...);
            }   
            goto finish;
        }
    }
    return err;
}
status_t IPCThreadState::talkWithDriver(bool doReceiver){
    binder_write_read bwr;
    const bool needRead = mIn.dataPosition()>=mIn.dataSize();
    const size_t outAvail = (!doReceive||needRead)?mOut.dataSize():0;
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();
    if(doReceive && needRead){
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    }else{
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr);
    return err;
}
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg){
    switch(cmd){
        case BINDER_WRITE_READ:{
            struct binder_write_read bwr;
            if(bwr.write_size>0){
                ret = binder_thread_write(proc, thread, bwr.write_buffer, ...);
            }
            if(bwr.read.size>0){
                ret = binder_thread_read(proc, thread, bwr.read_buffer, bwr.read_size, ...);
            }
        }
    }
}
```

Service 端如何和 binder 交互的？

```c++
virtual bool threadLoop(){
	IPCThreadState::self()->joinThreadPool(mIsMain);
    return false;
}
void IPCThreadState::joinThreadPool(bool isMain){
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER:BC_REGISTER_LOOPER);
    status_t result;
    do{
        result = getAndExecuteCommand();
    }while(result != -ECONNREFUSED && result != -EBADF);
    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
status_t IPCThreadState::getAndExecuteCommand(){
    // 1. 从驱动读取请求
    talkWithDriver();
    int32_t cmd = mIn.readInt32();
    // 2. 处理请求
    executeCommand(cmd);
}
status_t IPCThreadState::executeCommand(int32_t cmd){
    switch((uint32_t)cmd){
        case BR_TRANSACTION:{
            binder_transaction_data tr;
            mIn.read(&tr, sizeof(tr));
            
            Parcel buffer, reply;
            buffer.ipcSetDataReference(...);
            
            sp<BBinder> b((BBinder*)tr.cookie);
            b->transact(tr.code, buffer, &reply, tr.flags);
            sendReply(reply, 0);
        }
    }
    return result;
}
status_t IPCThreadState::sendReply(const Parcel& reply, uint32_t flags){
    writeTransactionData(BC_REPLY, flags, -1, 0, reply, &statusBuffer);
    return waitForResponse(NULL, NULL);
}
status_t BBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags){
    switch(code){
        case PING_TRANNSACTION:
            reply->writeInt32(pingBinder());
            break;
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }
    return err;
}
status_t onTransact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0){
    JNIEnv* env = javavm_to_jnienv(mVM);
    jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact, code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
    return res!=JNI_FALSE?NO_ERROE:UNKNOWN_TRANSACTION;
}
private boolean execTransact(int code, long dataObj, long replyObj, int flags){
    Parcel data = Parcel.obtain(dataObj);
    Parcel reply = Parcel.obtain(replyObj);
    boolean res = onTransact(code, data, reply, flags);
    reply.recycle();
    data.recycle();
    return res;
}
public static abstract class Stub extends android.os.Binder implements IRemoteCaller{
    boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags){
        switch(code){
            data.enforceInterface(descriptor);
                ICallback _arg0 = ICallback.Stub.asInterface(data.readStrongBinder());
                this.publishBinder(_arg0);
                reply.writeNoException();
                return true;
        }
    }
}
```

![](https://i.loli.net/2020/03/28/1ZbMj2fUiX8BGc7.png)