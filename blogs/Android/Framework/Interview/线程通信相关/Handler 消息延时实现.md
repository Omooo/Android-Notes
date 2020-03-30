---
Handler 的消息延时是怎么实现的？
---

1. 消息延时是做了什么特殊处理？
2. 是发送延时了，还是消息延时处理了？
3. 延时精度怎么样？

```java
boolean sendMessageDelayed(Message msg, long delayMillis){
    if(delayMillis<0){
        delayMillis=0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(Message msg, long uptimeMillis){
    MessageQueue queue = mQueue;
    return enqueueMessage(queue, msg, uptimeMillis);
}
boolean enqueueMessage(Message msg, long when){
    msg.when = when;
    Message p = mMessage, prew;
    if(p==null||when==0||when<p.when){
        msg.mext = p;
        mMessage = msg;
    }else{
        for(;;){
            prew = p;
            p = p.next;
            if(p==null||when<p.when){
                break;
            }
        }
        msg.next = p;
        prev.next = msg;
    }
    nativeWake(mPtr);
}
Message next(){
    for(int nextPollTimeoutMillis = 0;;){
        nativePollOnce(ptr, nextPollTimeoutMillis);
        final long now = SystemClock.uptimeMillis();
        Message msg = mMessages;
        if(msg != null){
            if(now<msg.when){
                nextPollTimeoutMillis = (int)Math.min(msg.when-now, Integer.MAX_VALUE);
            }else{
                mMessage = msg.next;
                msg.next = null;
                return msg;
            }
        }else{
            nextPollTimeoutMillis = -1;
        }
    }
}
```

#### 总结

1. 消息队列按消息触发时间排序
2. 设置 epoll_wait 的超时时间，使其在特定时间唤醒
3. 关于延时精度