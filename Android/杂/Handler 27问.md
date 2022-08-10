Handler二十七问｜你真的了解我吗？
https://mp.weixin.qq.com/s/63zHksIEEgm3PFNLtxIUrg

![概要图](https://mmbiz.qpic.cn/mmbiz_png/jfNJsrA21FKTkB9LFWLia2dKNRl0XiaR0tKzhnPZPvaKhOJ3jkEiaAd3y1F0KGuN7YEXrB5xHOHlgtoX8fMnia3pLg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


### Handler被设计出来的原因？有什么用？


一种东西被设计出来肯定就有它存在的意义，而Handler的意义就是切换线程。

作为Android消息机制的主要成员，它管理着所有与界面有关的消息事件，常见的使用场景有：

- 跨进程之后的界面消息处理。

  比如Activity的启动，就是AMS在进行进程间通信的时候，通过Binder线程 将消息发送给ApplicationThread的消息处理者Handler，然后再将消息分发给主线程中去执行。

-  网络交互后切换到主线程进行UI更新。

  当子线程网络操作之后，需要切换到主线程进行UI更新。


总之一句话，Hanlder的存在就是为了解决在子线程中无法访问UI的问题。

**为什么建议子线程不访问（更新）UI？**

因为Android中的UI控件不是线程安全的，如果多线程访问UI控件那还不乱套了。

**那为什么不加锁呢？**

会降低UI访问的效率。本身UI控件就是离用户比较近的一个组件，加锁之后自然会发生阻塞，那么UI访问的效率会降低，最终反应到用户端就是这个手机有点卡。

太复杂了。本身UI访问时一个比较简单的操作逻辑，直接创建UI，修改UI即可。如果加锁之后就让这个UI访问的逻辑变得很复杂，没必要。

所以，Android设计出了 单线程模型 来处理UI操作，再搭配上Handler，是一个比较合适的解决方案。

**子线程访问UI的 崩溃原因 和 解决办法？**

崩溃发生在`ViewRootImpl`类的`checkThread`方法中：
```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

其实就是判断了当前线程 是否是 ViewRootImpl创建时候的线程，如果不是，就会崩溃。

而ViewRootImpl创建的时机就是界面被绘制的时候，也就是onResume之后，所以如果在子线程进行UI更新，就会发现当前线程（子线程）和View创建的线程（主线程）不是同一个线程，发生崩溃。

解决办法有三种：

1、在新建视图的线程进行这个视图的UI更新，主线程创建View，主线程更新View。

2、在ViewRootImpl创建之前进行子线程的UI更新，比如onCreate方法中进行子线程更新UI。

3、子线程切换到主线程进行UI更新，比如Handler、view.post方法。

### MessageQueue是干嘛呢？用的什么数据结构来存储数据？


看名字应该是个队列结构，队列的特点是什么？先进先出，一般在队尾增加数据，在队首进行取数据或者删除数据。

那Hanlder中的消息似乎也满足这样的特点，先发的消息肯定就会先被处理。但是，Handler中还有比较特殊的情况，比如延时消息。

延时消息的存在就让这个队列有些特殊性了，并不能完全保证先进先出，而是需要根据时间来判断，所以Android中采用了链表的形式来实现这个队列，也方便了数据的插入。

来一起看看消息的发送过程，无论是哪种方法发送消息，都会走到sendMessageDelayed方法。


```java
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
sendMessageDelayed方法主要计算了消息需要被处理的时间，如果delayMillis为0，那么消息的处理时间就是当前时间。



然后就是关键方法enqueueMessage。


```java
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }

        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

不懂得地方先不看，只看我们想看的：



首先设置了Message的when字段，也就是代表了这个消息的处理时间。

然后判断当前队列是不是为空，是不是即时消息，是不是执行时间when大于表头的消息时间，满足任意一个，就把当前消息msg插入到表头。

否则，就需要遍历这个队列，也就是链表，找出when小于某个节点的when，找到后插入。


好了，其他内容暂且不看，总之，插入消息就是通过消息的执行时间，也就是when字段，来找到合适的位置插入链表。



具体方法就是通过死循环，使用快慢指针p和prev，每次向后移动一格，直到找到某个节点p的when大于我们要插入消息的when字段，则插入到p和prev之间。或者遍历到链表结束，插入到链表结尾。



所以，MessageQueue就是一个用于存储消息、用链表实现的特殊队列结构。



### 延迟消息是怎么实现的？


总结上述内容，延迟消息的实现主要跟消息的统一存储方法有关，也就是上文说过的enqueueMessage方法。



无论是即时消息还是延迟消息，都是计算出具体的时间，然后作为消息的when字段进程赋值。



然后在MessageQueue中找到合适的位置（安排when小到大排列），并将消息插入到MessageQueue中。



这样，MessageQueue就是一个按照消息时间排列的一个链表结构。



MessageQueue的消息怎么被取出来的？


刚才说过了消息的存储，接下来看看消息的取出，也就是queue.next方法。


```java
Message next() {
    for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
           }
    }
}
```

奇怪，为什么取消息也是用的死循环呢？



其实死循环就是为了保证一定要返回一条消息，如果没有可用消息，那么就阻塞在这里，一直到有新消息的到来。



其中，nativePollOnce方法就是阻塞方法，nextPollTimeoutMillis参数就是阻塞的时间。



那什么时候会阻塞呢？两种情况：



1、有消息，但是当前时间小于消息执行时间，也就是代码中的这一句：
```java
if (now < msg.when) {
    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
}
```
这时候阻塞时间就是消息时间减去当前时间，然后进入下一次循环，阻塞。



2、没有消息的时候，也就是上述代码的最后一句：

if (msg != null) {}
    else {
    // No more messages.
    nextPollTimeoutMillis = -1;
    }


-1就代表一直阻塞。



### MessageQueue没有消息时候会怎样？阻塞之后怎么唤醒呢？说说pipe/epoll机制？


接着上文的逻辑，当消息不可用或者没有消息的时候就会阻塞在next方法，而阻塞的办法是通过pipe/epoll机制。



epoll机制是一种IO多路复用的机制，具体逻辑就是一个进程可以监视多个描述符，当某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作，这个读写操作是阻塞的。在Android中，会创建一个Linux管道（Pipe）来处理阻塞和唤醒。



当消息队列为空，管道的读端等待管道中有新内容可读，就会通过epoll机制进入阻塞状态。

当有消息要处理，就会通过管道的写端写入内容，唤醒主线程。

### 同步屏障和异步消息是怎么实现的？


其实在Handler机制中，有三种消息类型：



同步消息。也就是普通的消息。

异步消息。通过setAsynchronous(true)设置的消息。

同步屏障消息。通过postSyncBarrier方法添加的消息，特点是target为空，也就是没有对应的handler。


这三者之间的关系如何呢？



正常情况下，同步消息和异步消息都是正常被处理，也就是根据时间when来取消息，处理消息。

当遇到同步屏障消息的时候，就开始从消息队列里面去找异步消息，找到了再根据时间决定阻塞还是返回消息。


也就是说同步屏障消息不会被返回，他只是一个标志，一个工具，遇到它就代表要去先行处理异步消息了。



所以同步屏障和异步消息的存在的意义就在于有些消息需要“加急处理”。



同步屏障和异步消息有具体的使用场景吗？


使用场景就很多了，比如绘制方法scheduleTraversals。


```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 同步屏障，阻塞所有的同步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 通过 Choreographer 发送绘制任务
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}


Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
msg.arg1 = callbackType;
msg.setAsynchronous(true);
mHandler.sendMessageAtTime(msg, dueTime);
```
在该方法中加入了同步屏障，后续加入一个异步消息MSG_DO_SCHEDULE_CALLBACK，最后会执行到FrameDisplayEventReceiver，用于申请VSYNC信号。



更多Choreographer相关内容可以看看这篇文章——https://www.jianshu.com/p/86d00bbdaf60


Message消息被分发之后会怎么处理？消息怎么复用的？


再看看loop方法，在消息被分发之后，也就是执行了dispatchMessage方法之后，还偷偷做了一个操作——recycleUnchecked。


```java
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // might block

        try {
            msg.target.dispatchMessage(msg);
        }

        msg.recycleUnchecked();
    }
}

//Message.java
private static Message sPool;
private static final int MAX_POOL_SIZE = 50;

void recycleUnchecked() {
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = UID_NONE;
    workSourceUid = UID_NONE;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

在recycleUnchecked方法中，释放了所有资源，然后将当前的空消息插入到sPool表头。



这里的sPool就是一个消息对象池，它也是一个链表结构的消息，最大长度为50。



那么Message又是怎么复用的呢？在Message的实例化方法obtain中：


```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

直接复用消息池sPool中的第一条消息，然后sPool指向下一个节点，消息池数量减一。



### Looper是干嘛呢？怎么获取当前线程的Looper？为什么不直接用Map存储线程和对象呢？


在Handler发送消息之后，消息就被存储到MessageQueue中，而Looper就是一个管理消息队列的角色。Looper会从MessageQueue中不断的查找消息，也就是loop方法，并将消息交回给Handler进行处理。



而Looper的获取就是通过ThreadLocal机制:


```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

通过prepare方法创建Looper并且加入到sThreadLocal中，通过myLooper方法从sThreadLocal中获取Looper。



### ThreadLocal运行机制？这种机制设计的好处？


下面就具体说说ThreadLocal运行机制。


```java
//ThreadLocal.java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
从ThreadLocal类中的get和set方法可以大致看出来，有一个ThreadLocalMap变量，这个变量存储着键值对形式的数据。



key为this，也就是当前ThreadLocal变量。

value为T，也就是要存储的值。


然后继续看看ThreadLocalMap哪来的，也就是getMap方法：



//ThreadLocal.java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

//Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;


原来这个ThreadLocalMap变量是存储在线程类Thread中的。



所以ThreadLocal的基本机制就搞清楚了。



在每个线程中都有一个threadLocals变量，这个变量存储着ThreadLocal和对应的需要保存的对象。



这样带来的好处就是，在不同的线程，访问同一个ThreadLocal对象，但是能获取到的值却不一样。



挺神奇的是不是，其实就是其内部获取到的Map不同，Map和Thread绑定，所以虽然访问的是同一个ThreadLocal对象，但是访问的Map却不是同一个，所以取得值也不一样。



这样做有什么好处呢？为什么不直接用Map存储线程和对象呢？



打个比方：



ThreadLocal就是老师。

Thread就是同学。

Looper（需要的值）就是铅笔。


现在老师买了一批铅笔，然后想把这些铅笔发给同学们，怎么发呢？两种办法：



1、老师把每个铅笔上写好每个同学的名字，放到一个大盒子里面去（map），用的时候就让同学们自己来找。

这种做法就是Map里面存储的是同学和铅笔，然后用的时候通过同学来从这个Map里找铅笔。



这种做法就有点像使用一个Map，存储所有的线程和对象，不好的地方就在于会很混乱，每个线程之间有了联系，也容易造成内存泄漏。



2、老师把每个铅笔直接发给每个同学，放到同学的口袋里（map），用的时候每个同学从口袋里面拿出铅笔就可以了。

这种做法就是Map里面存储的是老师和铅笔，然后用的时候老师说一声，同学只需要从口袋里拿出来就行了。



很明显这种做法更科学，这也就是ThreadLocal的做法，因为铅笔本身就是同学自己在用，所以一开始就把铅笔交给同学自己保管是最好的，每个同学之间进行隔离。



### 还有哪些地方运用到了ThreadLocal机制？


比如：Choreographer。


```java
public final class Choreographer {

    // Thread local storage for the choreographer.
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
            if (looper == Looper.getMainLooper()) {
                mMainInstance = choreographer;
            }
            return choreographer;
        }
    };

    private static volatile Choreographer mMainInstance;
```
Choreographer主要是主线程用的，用于配合 VSYNC中断信号。



所以这里使用ThreadLocal更多的意义在于完成线程单例的功能。



### 可以多次创建Looper吗？


Looper的创建是通过Looper.prepare方法实现的，而在prepare方法中就判断了，当前线程是否存在Looper对象，如果有，就会直接抛出异常：


```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

所以同一个线程，只能创建一个Looper，多次创建会报错。



### Looper中的quitAllowed字段是啥？有什么用？


按照字面意思就是是否允许退出，我们看看他都在哪些地方用到了：


```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }
    }
}
```
哦，就是这个quit方法用到了，如果这个字段为false，代表不允许退出，就会报错。



但是这个quit方法又是干嘛的呢？从来没用过呢。还有这个safe又是啥呢？



其实看名字就差不多能了解了，quit方法就是退出消息队列，终止消息循环。



首先设置了mQuitting字段为true。

然后判断是否安全退出，如果安全退出，就执行removeAllFutureMessagesLocked方法，它内部的逻辑是清空所有的延迟消息，之前没处理的非延迟消息还是需要取处理，然后设置非延迟消息的下一个节点为空（p.next=null）。

如果不是安全退出，就执行removeAllMessagesLocked方法，直接清空所有的消息，然后设置消息队列指向空（mMessages = null）。


然后看看当调用quit方法之后，消息的发送和处理：


```java
//消息发送
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
    }
```
当调用了quit方法之后，mQuitting为true，消息就发不出去了，会报错。

再看看消息的处理，loop和next方法：


```java
Message next() {
    for (;;) {
        synchronized (this) {
            if (mQuitting) {
                dispose();
                return null;
            }
        }  
    }
}


public static void loop() {
    for (;;) {
        Message msg = queue.next();
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
    }
}
```
很明显，当mQuitting为true的时候，next方法返回null，那么loop方法中就会退出死循环。



那么这个quit方法一般是什么时候使用呢？



主线程中，一般情况下肯定不能退出，因为退出后主线程就停止了。所以是当APP需要退出的时候，就会调用quit方法，涉及到的消息是EXIT_APPLICATION，大家可以搜索下。

子线程中，如果消息都处理完了，就需要调用quit方法停止消息循环。

### Looper.loop方法是死循环，为什么不会卡死（ANR）？


关于这个问题，强烈建议看看Gityuan的回答：

https://www.zhihu.com/question/34652589



我大致总结下：



1、主线程本身就是需要一只运行的，因为要处理各个View，界面变化。所以需要这个死循环来保证主线程一直执行下去，不会被退出。

2、真正会卡死的操作是在某个消息处理的时候操作时间过长，导致掉帧、ANR，而不是loop方法本身。

3、在主线程以外，会有其他的线程来处理接受其他进程的事件，比如Binder线程（ApplicationThread），会接受AMS发送来的事件。

4、在收到跨进程消息后，会交给主线程的Hanlder再进行消息分发。所以Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message时则采用相应措施，比如收到msg=H.LAUNCH_ACTIVITY，则调用ActivityThread.handleLaunchActivity()方法，最终执行到onCreate方法。

5、当没有消息的时候，会阻塞在loop的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生。所以死循环也不会特别消耗CPU资源。

Message是怎么找到它所属的Handler然后进行分发的？


在loop方法中，找到要处理的Message，然后调用了这么一句代码处理消息：



msg.target.dispatchMessage(msg);


所以是将消息交给了msg.target来处理，那么这个target是啥呢？



找找它的来头：


```
//Handler
private boolean enqueueMessage(MessageQueue queue,Message msg,long uptimeMillis) {
    msg.target = this;

    return queue.enqueueMessage(msg, uptimeMillis);
}
```

在使用Hanlder发送消息的时候，会设置msg.target = this，所以target就是当初把消息加到消息队列的那个Handler。



Handler 的 post(Runnable) 与 sendMessage 有什么区别


Hanlder中主要的发送消息可以分为两种：



post(Runnable)

sendMessage
```
public final boolean post(@NonNull Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
通过post的源码可知，其实post和sendMessage的区别就在于：



post方法给Message设置了一个callback。



那么这个callback有什么用呢？我们再转到消息处理的方法dispatchMessage中看看：


```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}
```

这段代码可以分为三部分看：



1、如果msg.callback不为空，也就是通过post方法发送消息的时候，会把消息交给这个msg.callback进行处理，然后就没有后续了。

2、如果msg.callback为空，也就是通过sendMessage发送消息的时候，会判断Handler当前的mCallback是否为空，如果不为空就交给Handler.Callback.handleMessage处理。

3、如果mCallback.handleMessage返回true，则无后续了。

4、如果mCallback.handleMessage返回false，则调用handler类重写的handleMessage方法。

所以post(Runnable) 与 sendMessage的区别就在于后续消息的处理方式，是交给msg.callback还是 Handler.Callback或者Handler.handleMessage。



Handler.Callback.handleMessage 和 Handler.handleMessage 有什么不一样？为什么这么设计？


接着上面的代码说，这两个处理方法的区别在于Handler.Callback.handleMessage方法是否返回true：



如果为true，则不再执行Handler.handleMessage。

如果为false，则两个方法都要执行。

那么什么时候有Callback，什么时候没有呢？这涉及到两种Hanlder的 创建方式：

```java
val handler1= object : Handler(){
    override fun handleMessage(msg: Message) {
        super.handleMessage(msg)
    }
}

val handler2 = Handler(object : Handler.Callback {
    override fun handleMessage(msg: Message): Boolean {
        return true
    }
})
```
常用的方法就是第1种，派生一个Handler的子类并重写handleMessage方法。而第2种就是系统给我们提供了一种不需要派生子类的使用方法，只需要传入一个Callback即可。



Handler、Looper、MessageQueue、线程是一一对应关系吗？


一个线程只会有一个Looper对象，所以线程和Looper是一一对应的。

MessageQueue对象是在new Looper的时候创建的，所以Looper和MessageQueue是一一对应的。

Handler的作用只是将消息加到MessageQueue中，并后续取出消息后，根据消息的target字段分发给当初的那个handler，所以Handler对于Looper是可以多对一的，也就是多个Hanlder对象都可以用同一个线程、同一个Looper、同一个MessageQueue。

总结：Looper、MessageQueue、线程是一一对应关系，而他们与Handler是可以一对多的。



ActivityThread中做了哪些关于Handler的工作？（为什么主线程不需要单独创建Looper）


主要做了两件事：



1、在main方法中，创建了主线程的Looper和MessageQueue，并且调用loop方法开启了主线程的消息循环。
```java
public static void main(String[] args) {

    Looper.prepareMainLooper();

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
2、创建了一个Handler来进行四大组件的启动停止等事件处理。
```java
final H mH = new H();

class H extends Handler {
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    public static final int RECEIVER                = 113;
    public static final int CREATE_SERVICE          = 114;
    public static final int STOP_SERVICE            = 116;
    public static final int BIND_SERVICE            = 121;
```
IdleHandler是啥？有什么使用场景？


之前说过，当MessageQueue没有消息的时候，就会阻塞在next方法中，其实在阻塞之前，MessageQueue还会做一件事，就是检查是否存在IdleHandler，如果有，就会去执行它的queueIdle方法。


```java
private IdleHandler[] mPendingIdleHandlers;

Message next() {
    int pendingIdleHandlerCount = -1;
    for (;;) {
        synchronized (this) {
            //当消息执行完毕，就设置pendingIdleHandlerCount
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }

            //初始化mPendingIdleHandlers
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            //mIdleHandlers转为数组
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 遍历数组，处理每个IdleHandler
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            //如果queueIdle方法返回false，则处理完就删除这个IdleHandler
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;
    }
}
```
当没有消息处理的时候，就会去处理这个mIdleHandlers集合里面的每个IdleHandler对象，并调用其queueIdle方法。最后根据queueIdle返回值判断是否用完删除当前的IdleHandler。



然后看看IdleHandler是怎么加进去的：


```java
Looper.myQueue().addIdleHandler(new IdleHandler() {  
@Override  
public boolean queueIdle() {  
    //做事情
    return false;    
}  
});

public void addIdleHandler(@NonNull IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}
```
ok，综上所述，IdleHandler就是当消息队列里面没有当前要处理的消息了，需要堵塞之前，可以做一些空闲任务的处理。



常见的使用场景有：启动优化。



我们一般会把一些事件（比如界面view的绘制、赋值）放到onCreate方法或者onResume方法中。但是这两个方法其实都是在界面绘制之前调用的，也就是说一定程度上这两个方法的耗时会影响到启动时间。



所以我们可以把一些操作放到IdleHandler中，也就是界面绘制完成之后才去调用，这样就能减少启动时间了。



但是，这里需要注意下可能会有坑。



如果使用不当，IdleHandler会一直不执行，比如在View的onDraw方法里面无限制的直接或者间接调用View的invalidate方法。



其原因就在于onDraw方法中执行invalidate，会添加一个同步屏障消息，在等到异步消息之前，会阻塞在next方法，而等到FrameDisplayEventReceiver异步任务之后又会执行onDraw方法，从而无限循环。



具体可以看看这篇文章：https://mp.weixin.qq.com/s/dh_71i8J5ShpgxgWN5SPEw


HandlerThread是啥？有什么使用场景？


直接看源码：


```java
public class HandlerThread extends Thread {
@Override
public void run() {
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
}
```
哦，原来如此。HandlerThread就是一个封装了Looper的Thread类。



就是为了让我们在子线程里面更方便的使用Handler。



这里的加锁就是为了保证线程安全，获取当前线程的Looper对象，获取成功之后再通过notifyAll方法唤醒其他线程，那哪里调用了wait方法呢？


```java
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }

    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```

就是getLooper方法，所以wait的意思就是等待Looper创建好，那边创建好之后再通知这边正确返回Looper。



IntentService是啥？有什么使用场景？


老规矩，直接看源码：


```java
public abstract class IntentService extends Service {

private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        stopSelf(msg.arg1);
    }
}

@Override
public void onCreate() {
    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}

@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```
理一下这个源码：



首先，这是一个Service。

并且内部维护了一个HandlerThread，也就是有完整的Looper在运行。

还维护了一个子线程的ServiceHandler。

启动Service后，会通过Handler执行onHandleIntent方法。

完成任务后，会自动执行stopSelf停止当前Service。


所以，这就是一个可以在子线程进行耗时任务，并且在任务执行后自动停止的Service。



BlockCanary使用过吗？说说原理


BlockCanary是一个用来检测应用卡顿耗时的三方库。



上文说过，View的绘制也是通过Handler来执行的，所以如果能知道每次Handler处理消息的时间，就能知道每次绘制的耗时了？那Handler消息的处理时间怎么获取呢？



再去loop方法中找找细节：


```java
public static void loop() {
    for (;;) {
        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
    }
}
```
可以发现，loop方法内有一个Printer类，在dispatchMessage处理消息的前后分别打印了两次日志。



那我们把这个日志类Printer替换成我们自己的Printer，然后统计两次打印日志的时间不就相当于处理消息的时间了？


```java
 Looper.getMainLooper().setMessageLogging(mainLooperPrinter);

    public void setMessageLogging(@Nullable Printer printer) {
        mLogging = printer;
    }
```

这就是BlockCanary的原理。



具体介绍可以看看作者的说明：

http://blog.zhaiyifan.cn/2016/01/16/BlockCanaryTransparentPerformanceMonitor/


说说Hanlder内存泄露问题。


这也是常常被问的一个问题，Handler内存泄露的原因是什么？



"内部类持有了外部类的引用，也就是Hanlder持有了Activity的引用，从而导致无法被回收呗。"



其实这样回答是错误的，或者说没回答到点子上。



我们必须找到那个最终的引用者，不会被回收的引用者，其实就是主线程，这条完整引用链应该是这样：



主线程 —> threadlocal —> Looper —> MessageQueue —> Message —> Handler —> Activity



具体分析可以看看我之前写的这篇文章：https://juejin.cn/post/6909362503898595342


利用Handler机制设计一个不崩溃的App？


主线程崩溃，其实都是发生在消息的处理内，包括生命周期、界面绘制。



所以如果我们能控制这个过程，并且在发生崩溃后重新开启消息循环，那么主线程就能继续运行。


```java
Handler(Looper.getMainLooper()).post {
    while (true) {
        //主线程异常拦截
        try {
            Looper.loop()
        } catch (e: Throwable) {
        }
    }
}
```
还有一些特殊情况处理，比如onCreate内发生崩溃，具体可以看看文章《能否让APP永不崩溃》。

https://juejin.cn/post/6904283635856179214


总结


大家应该可以发现，有一个问题常被问，但是全篇都没有提，那就是：Hanlder机制的运行原理。



之所以不提这个问题，是因为要回答好这个问题需要大量知识储备，希望屏幕前的你在读完这篇之后，再结合自己的知识库，形成自己的“完美答案”。



Hanlder，I Got it！



参考
《Android开发艺术探索》

https://www.zhihu.com/question/34652589

https://segmentfault.com/a/1190000003063859

https://juejin.cn/post/6844904150140977165

https://juejin.cn/post/6893791473121280013
