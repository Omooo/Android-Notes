---
Surface 的绘制原理
---

1. Surface 绘制的 buffer 是怎么来的？
2. buffer 绘制完了又是怎么提交的？

```java
private void performTraversals(){
    performMeasure();
    performLayout();
    performDraw();
}
private void draw(boolean fullRedrawNeeded){
    Surface surface = mSurface;
    drawSoftware(surface, mAttachInfo, ..., dirty);
}
private boolean drawSoftware(Surface surface, ...){
    final Canvas canvas;
    canvas = mSurface.lockCanvas(dirty);
    mView.draw(canvas);
    surface.unlockCanvasAndPost(canvas);
}
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz, jlong nativeObject, jobject canvasObj, jobject dirtyRectObj){
    sp<Surface> surface(reinterpert_cast<Surface *>(nativeObject));
    ANativeWindow_Buffer outBuffer;
    surface->lock(&outBuffer, dirtyRectPtr);
    SkBitmap bitmap;
    bitmap.setPixels(outBuffer.bits);
    Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
    nativeCanvas->setBitmap(bitmap);
    sp<Surface> lockedSurface(surface);
    return (jlong)lockedSurface.get();
}
```

每次绘制都要重新申请一块 buffer，它是怎么申请的？

```c++
status_t Surface::lock(ANativeWindow_Buffer* outBuffer, ...){
	ANativeWindowBuffer* out;
    dequeueBuffer(&out, &fenceFd);
    sp<GraphicBuffer> backBuffer(GraphicBuffer::getSelf(out));
    void* vaddr;
    status_t res = backBuffer->lockAsync(..., &vaddr, fenceFd);
    mLockedBuffer = backBuffer;
    outBuffer->bits = vaddr;
}
int Surface::dequeueBuffer(android_native_buffer_t** buffer, ...){
    int buf = -1;
    status_t result = mGraphicBufferProduct->dequeueBuffer(&buf, ...);
    sp<GraphicBuffer>& gbuf(mSlot[buf].buffer);
    if((result&IGraphicBufferProduct::BUFFER_NEEDS_REALLOCATION)||gbuf == 0){
        result = mGraphicBufferProduct->requestBuffer(buf, &gbuf);
    }
    *buffer = gbuf.get();
    return OK;
}
```

Buffer 是怎么提交的？

```c++
public void unlockCanvasAndPost(Canvas canvas){
	synchronized(mLock){
        unlockSwCanvasAndPost(canvas);
    }
}
private void unlockSwCanvasAndPost(Canvas canvas){
    nativeUnlockCanvasAndPost(mLockedObject, canvas);
    nativeRelease(mLockedObject);
    mLockedObject = 0;
}
static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz, jlong nativeObject, jobject canvasObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
    nativeCanvas->setBitmap(SkBitmap());
    surface->unlockAndPost();
}
status_t Surface::unlockAndPost(){
    mLockedBuffer->unlockAsync(...);
    queueBuffer(mLockedBuffer.get(), ...);
    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = 0;
    return err;
}
int Surface::queueBuffer(android_native_buffer_t* buffer, ...){
    int i = getSlotFromBufferLocked(buffer);
    mGraphicBufferProducer->queueBuffer(i, ...);
}
```

![](https://i.loli.net/2020/03/26/l1QyIsgBtPSvLpF.png)