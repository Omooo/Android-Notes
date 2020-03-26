---
Surface 跨进程传递原理
---

1. 怎么理解 surface，它是一块 buffer 嘛？
2. 如果是，surface 跨进程传递怎么带上这个 buffer？
3. 如果不是，那么 surface 跟 buffer 又是什么关系？
4. surface 到底是怎么跨进程传递的？

```java
public class Surface implements Parcelable{
    long mNativeObject;
    private final Canvas mCanvas = new CompatibleCanvas();
}

public void writeToParcel(Parcel dest, int flags){
    synchronized(mLock){
        dest.writeString(mName);
        nativeWriteToParcel(mNativeObject, dest);
    }
}
public void readFromParcel(Parcel source){
    synchronized(mLock){
        mName = source.readString();
        setNativeObjectLocked(nativeReadFromParcel(mNativeObject, source));
    }
}

static void nativeWriteToParcel(JNIEnv* env, jclass clazz, jlong nativeObject, jobject parcelObj){
    Parcel* parcel = parcelForJavaObject(env, parcelObj);
    sp<Surface> self(reinterpert_cast<Surface*>(nativeObject));
    parcel->writeStrongBinder(IInterface::asBinder(self->getIGraphicBufferProducer()));
}
Parcel* parcelForJavaObject(JNIEnv* env, jobject obj){
    Parcel* p = env->GetLongField(obj, gParcelOffsets.mNativePtr);
    return p;
}
sp<IGraphicBufferProduct> getIGraphicBufferProducer() const{
    return mProducer;
}

static jlong nativeReadFromParcel(JNIEnv* env, jclass clazz, jlong nativeObject, jobject parcelObj){
    Parcel* parcel = parcelForJavaObject(env, parcelObj);
    sp<Surface> self(reinterpret_cast<Surface *>(nativeObject));
    sp<IBinder> binder(parcel->readStrongBinder());
    sp<IGraphicBufferProducer> gbp(interface_cast<IGraphicBufferProducer>(binder));
    sp<Surface> sur = new Surface(gbp, true);
    return jlong(sur.get());
}
```

#### Activity 的 surface 是怎么跨进程传递的？

```java
private void performTraversals(){
	final View host = mView;
    if(mFirst){
        relayoutResult = relayoutWindow(params, ...);
    }
}
int relayoutWindow(WindowManager.LayoutParams params, ...){
    return mWindowSession.relayout(mWindow, ..., mSurface);
}
// WMS
public int relayout(IWindow window, ..., Surface outSurface){
    int res = mService.relayoutWindow(this, window, ..., outSurface);
    return res;
}
public int relayoutWindow(Session session, ..., Surface outSurface){
    SurfaceControl surfaceControl = winAnimator.createSurfaceLocked();
    outSurface.copyForm(surfaceControl);
}
public void copyForm(SurfaceControl other){
    long surfaceControlPtr = other.mNativeObject;
    long newNativeObject = nativeCreateFromSurfaceControl(surfaceControlPtr);
    setNativeObjectLocked(newNativeObject);
}
jlong nativeCreateFromSurfaceControl(JNIEnv* env, jclass clazz, ...){
    sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj));
    sp<Surface> surface(ctrl->getSurface());
    return reinterpret_cast<jlong>(surface.get());
}
sp<Surface> SurfaceControl::getSurface() const{
    if(mSurfaceData == 0){
        mSurfaceData = new Surface(mGraphicBufferProducer, false);
    }
    return mSurfaceData;
}
```

#### 总结

1. Surface 的本质是 GraphicBufferProducer，而不是 buffer
2. Surface 跨进程传递，本质就是 GraphicBufferProducer 的传递

Surface 不是一块 buffer，而是一个壳子，里面包含了一个能够生产 buffer 的 Binder 对象，也就是 GBP。Surface 跨进程传递，只是传递 GBP 对象。