---
说说 Android 的 UI 刷新机制
---

1. 丢帧一般是什么原因引起的？
2. Android 刷新频率 60 帧/秒，每隔 16 ms 调 onDraw 绘制一次？
3. onDraw 完之后屏幕会马上刷新吗？
4. 如果界面没有重绘，还会每隔 16ms 刷新屏幕吗？
5. 如果在屏幕快要刷新的时候才去 onDraw 绘制会丢帧吗？

```java
public void requestLayout(){
	scheduleTraversals();
}
void scheduleTraversals(){
    mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
    if(!mTraversalScheduled){
        mTraversalScheduled = true;
        mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}
// mTraversalRunnable
void doTraversal(){
    if(mTraversalScheduled){
        mTraversalScheduled = false;
        performTraversals();
    }
}
```

```java
private void postCallbackDelayedInternal(int callbackType, ){
    mCallbackQueues[callbackType].addCallbackLocked(dueTime, ...);
    scheduleFrameLocked(now);
}
private void scheduleFrameLocked(long now){
    if(isRunningOnLooperThreadLocked()){
        scheduleVsyncLocked();
    }else{
        Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
        msg.setAsynchronous(true);
        mHandler.sendMEssageAtFrontOfQueue(msg);
    }
}
```

```java
class FrameDisplayEventReceiver extends DisplayEventReceiver{
	public void onVsync(long timestampNanos, int builtInDisplayId, int frame){
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos/TimeUtils.NANOS_PER_MS);
    }
    public void run(){
        doFrame(mTimestampNanos, mFrame);
    }
}
// Choreographer#doFrame
void doFrame(long frameTimeNanos, int frame){
    long intendedFrameTimeNanos = frameTimeNanos;
    startNanos = System.nanoTime();
    final long jitterNanos = startNanos - frameTimeNanos;
    if (jitterNanos >= mFrameIntervalNanos) {
        final long skippedFrames = jitterNanos / mFrameIntervalNanos;
        if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  " + "The application may be doing too much work on its main thread.");
        }
        final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
        frameTimeNanos = startNanos - lastFrameOffset;
    }
    
    // 处理 Callback
    doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
}
void doCallback(int callbackType, long frameTimeNanos){
    CallbackRecord callbacks;
    callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(...);
    for (CallbackRecord c = callbacks; c != null; c = c.next) {
        c.run(frameTimeNanos);
    }
}
```

![](https://i.loli.net/2020/03/25/GfDFqBLMAvKbNdn.png)

```java
private void scheduleVsyncLocked(){
    mDisplayEventReceiver.scheduleVsync();
}
```

```c++
status_t NativeDisplayEventReceiver::scheduleVsync(){
	status_t status = mReceiver.requestNextVsync();
}
status_t DisplayEventReceiver::requestNextVsync(){
    mEventConnection->requestNextVsync();
}
DisplayEventReceiver::DisplayEventReceiver(){
    sp<ISurfaceComposer> sf(ComposerService::slef());
    if(sf!=null){
        mEventConnection = sf->createDisplayEventConnection();
        mDataChannel = mEventConnection->getDataChannel();
    }
}
sp<BitTube> EventThread::Connection::getDataChannel() const{
    return mChannel;
}
```

创建 Connection，实现是在 SurfaceFlinger 里：

```c++
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection(){
    return mEventThread->createEventConnection();
}
sp<EventThread::Connection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}
EventThread::Connection::Connection(const sp<EventThread>&eventThread):count(-1), mEventThread(eventThread), mChannel(new BitTubr()) {
    
}
void EventThread::Connection::onFirstRef(){
    mEventThread->registerDisplayEventConnection(this);
}
status_t EventThread::registerDisplayEventConnection(const sp<EventThread::Connection>&connection){
    mDisplayEventConnections.add(connection);
    mCondition.broadcast();
}
```

创建 EventThread:

```c++
void SurfaceFlinger::init(){
	sp<VSYNCSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync, ...);
	mEventThread = new EventThread(vsyncSrc);
}
```

```c++
bool EventThread::threadLoop(){
	DisplayEventReceiver::Event event;
	Vector<sp<EventThread::Connection>> signalConnections;
	signalConnections = waitForEvent(&event);
	const size_t count = signalConnection,size();
	for(size_t i = 0;i<count:i++){
		const sp<Connection>&conn(signalConnections[i]);
		conn->postEvent(event);
	}
	return true;
}
Vector<sp<EventThread::Connection>> EventThread::waitForEvent(...){
    Vector<sp<EventThread::Connectiob>> signalConnections;
    do{
        // 看是否已经有 vsync 信号来了
        // 如果有的话，就准备 connection 列表返回
        // 如果没有的话，就等待 vsync 信号
    }while(signalConnections.isEmpty());
    return signalConnections;
}
```

```c++
void EventThread::requestNextVsync(const sp<EventThread::Connection>&connection){
    if(connection->count<0){
        connection->count=0;
        mCondition.broadcast();
    }
}
```

```c++
status_t EventThread::Connection::postEvent(const DisplayEventReceiver::Event& event){
	ssize_t size = DisplayEventReceiver::sendEvents(mChannel, &event, 1);
	return size<0?status_t(size):status_t(NO_ERROR);
}
ssize_t size DisplayEventReceiver::sendEvents(const sp<BitTube> &dataChannel, Event const* events, size_t count) {
    return BitTube:sendObjects(dataChannel, events, count);
}
ssize_t BitTube::sendObjects(const sp<BitTube> &tube, ...){
    const char* vaddr = reinterpret_cast<const char*>(events);
    sszie_t size = tube->write(vaddr, count*objSize);
    return size<0?size:size/static_cast<sszie_t>(objSize);
}
```

BitTube 是在 SurfaceFlinger 创建的，读的一端是如何传递给应用进程呢？

```c++
DisplayEventReceiver::DisplayEventReceiver(){
	sp<ISrufaceComposer> sf(ComposerService::getComposerService());
    mEventConnection = sf->createDisplayEventConnection();
    mDataChannel = mEventConnection->getDataChannel();
}
virtual sp<BitTube> getDataChannel() const{
    Parcel data, reply;
    data.writeInterfaceToken(IDisplayEventConnection::getInterfaceDescriptor());
    remote()->transact(GET_DATA_CHANNEL, data, &reply);
    return new BitTube(reply);
}
sp<BitTube> EventThread::Connection::getDataChannel() const{
    return mChannel;
}
```

应用进程如何监听读的描述符呢？

```java
private Choreographer(Looper looper){
	mDisplayEventReceiver = new FrameDisplayEventReceiver(looper);
}
public DisplayEventReceiver(Looper looper){
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), ...);
}
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak, ...){
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(...);
    status_t status = receiver->initialize();
    return reinterpret_cast<jlong>(receiver.get());
}
status_t NativeDisplayEventReceiver::initialize(){
    mMessageQueue->getLooper()->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT, this, NULL);
    rwturn OK;
}
DisplayEventReceiver::DisplayEventReceiver(){
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    mEventConnection = sf->createisplayEventConnection();
    mDataChannel = mEventConnection->getDataChannel();
}
int DisplayEventReceiver::getFd() const{
    return mDataChannel->getFd();
}
int BitTube::getFd() const{
    return mReceiverFd;
}
int Looper::addFd(int fd, int ident, int events, ...){
    Request request:
    request.fd = fd;
    struct epoll_event eventItem:
    request.initEventItem(&eventItem);
    int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, &eventItem);
    mRequests.add(fd, request);
}
```

Looper 检测 fd 的读事件：

```java
int Looper::pollInner(int timeoutMillis){
    int eventCount = epoll_wait(mEpollFd, eventItems, ...);
    for(int i = 0;i<eventCount;i++){
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItem[i].events;
        if(fd == mWakeEventFd){
            
        }else{
            if(epollEvent&EPOLLIN)events|=EVENT_INPUT;
            pustResponse(events, mRequest.valueAt(requestIndex));
        }
    }
    // 统一处理 Response
    for(size_t i=0;i<mResponse.size();i++){
        Response& response = mResponse.editItemAt(i);
        int fd = response.request.fd;
        int events = response.events;
        void* data = response.request.data;
        int callbackResult = response.request.callback->handleEvent(fd, events, data);
        if(callbackResult==0){
            removeFd(fd, response.request.seq);
        }
        response.request.callback.clear();
        result = POLL_CALLBACK;
    }
    return result;
}
int NativeDisplayEventReceiver::handleEvent(int receiveFd, int events, ...){
    if(processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, ...)){
        mWaitingForVsync = false;
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }
    return 1;
}
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, ...){
    env->CallVoidMethod(receiverObj.get(), gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, ...);
}
void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame){
    onVsync(timestampNanos, buildInDisplayId, frame);
}
```

#### 总结

1. Vsync 的原理是怎样的？
2. Choreographer 的原理是怎样的？
3. UI 刷新的大致流程，应用和 SurfaceFlinger 的通信过程？