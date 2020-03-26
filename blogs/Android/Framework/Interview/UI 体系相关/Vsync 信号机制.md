---
Vsync 信号机制
---

1. Vsync 信号的生成机制
2. Vsync 在 SurfaceFlinger 中的分发流程
3. Vsync 信号的分发原理

![](https://i.loli.net/2020/03/26/XIw6DvOa2MTRKoj.png)

```c++
void SurfaceFlinger::init(){
    sp<VsyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync, vsyncPhaseOffsetNs, true, "app");
    mEventThread = new EventThread(vsyncSrc);
    sp<VsyncSource> sfVsyncSrc = new DisSyncSource(&mPrimaryDispSync, sfVsyncPhaseOffsetNs, true, "sf");
    mSFEventThread = new EventThread(sfVsyncSrc);
    mHwc = new HWComposer(this, *static_cast<HWCompiser::EventHandler *>(this));
}
// 主要用于生成 Vsync 信号
HWComposer::HWComposer(const sp<SurfaceFlinger>& flinger, ...){
    bool needVsyncThread = true;
    loadHwcModule();
    if(mHwc){
        mCBContext->procs.vsync = &hook_vsync;
        needVsyncThread = false;
    }
    if(needVsyncThread){
        mVsyncThread = new VsyncThread(*this);
    }
}
// hook_vsync
void HWComposer::vsync(int disp, int64_t timestamp){
    mEventHandler.onVSyncReceived(disp, timestamp);
}
// 软件分发
bool HWComposer::VSyncThread::threadLoop(){
    clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &spec, NULL);
    mHwc.mEventHandler.onVSyncReceived(0, next_vsync);
    return true;
}

void SurfaceFlinger::onVSyncReceived(int type, nsece_t timestamp){
    mPrimaryDispSync.addResyncSample(timestamp);
}
bool DispSync::addResyncSample(nsecs_t timestamp){
    mResyncSamples[idx] = timestamp;
    updateModelLocked();
}
void DispSync::updateModelLocked(){
    mThread->updateModel(mPeriod, mPhase);
}
DispSync::DispSync():mThread(new DispSyncThread()){
    mThread->run("DispSync", ...);
}
// 工作线程的处理流程
virtual bool threadLoop(){
    while(true){
        if(mPeriod == 0){
            mCond.wait(mMutex);
            continue;
        }
        now = systemTime(SYSTEM_TIME_MONOTONIC);
        callbackInvocations = gatherCallbackInvocationsLocked(now);
        fireCallbackInvocations(callbackInvocations);
    }
    return false;
}
```

```c++
void EventThread::onVSyncEvent(nsecs_t timestamp){
    mVSyncEvent[0].header.timestamp = timestamp;
    mCondition.broadcast();
}
Vector<sp<EventThread::Connection>> EventThread::waitForEvent(...){
    do{
        // 检查 mVSyncEvent 数组，看有没有 VSync 触发
        // 如果有就遍历所有注册的 connection
        // 如果 count>=0 就加到 signalConnections 里，加的时候重置为 -1
        if(!timestamp){
            if(waitForVSync){
                mCondition.waitRelative(mLock, timeout);
            }
        }
    }while(signalConnections.isEmpty());
    return signalConnections;
}
```

```c++
status_t EventThread::Connection::postEvent(...){
	DisplayEventReceiver::sendEvents(mChannel, &event, 1);
}
ssize_t DisplayEventReceiver::sendEvents(const sp<BitTube>&dataChannel, Event const* events, size_t count){
    return BitTube::sendObjects(dataChannel, events, count);
}
ssize_t BitTube::sendObjects(const sp<BitTube>&tube, void const* events, size_t count, size_t objSize){
    const char* vaddr = reinterpret_cast<const char*>(events);
    sszie_t size = tube->write(vaddr, count*objSize);
}
```

![](https://i.loli.net/2020/03/26/gsG1Jlv8cxmCzIE.png)