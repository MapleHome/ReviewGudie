# 消息机制
## Handler 机制
Handler 有两个主要用途：（1）安排 Message 和 runnables 在将来的某个时刻执行; （2）将要在不同于自己的线程上执行的操作排入队列。(在多个线程并发更新UI的同时保证线程安全。)

Android 规定访问 UI 只能在主线程中进行，因为 Android 的 UI 控件不是线程安全的，多线程并发访问会导致 UI 控件处于不可预期的状态。为什么系统不对 UI 控件的访问加上锁机制？缺点有两个：加锁会让 UI 访问的逻辑变得复杂；其次锁机制会降低 UI 访问的效率。如果子线程访问 UI，那么程序就会抛出异常。ViewRootImpl 对UI操作做了验证，这个验证工作是由 ViewRootImpl的 ``checkThread`` 方法完成：

``ViewRootImpl.java``
```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

- Message：Handler 接收和处理的消息对象
- MessageQueue：Message 的队列，先进先出，每一个线程最多可以拥有一个
- Looper：消息泵，是 MessageQueue 的管理者，会不断从 MessageQueue 中取出消息，并将消息分给对应的 Handler 处理，每个线程只有一个 Looper。

Handler 创建的时候会采用当前线程的 Looper 来构造消息循环系统，需要注意的是，线程默认是没有 Looper 的，直接使用 Handler 会报错，如果需要使用 Handler 就必须为线程创建 Looper，因为默认的 UI 主线程，也就是 ActivityThread，ActivityThread 被创建的时候就会初始化 Looper，这也是在主线程中默认可以使用 Handler 的原因。

## 工作原理
### ThreadLocal
ThreadLocal 是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，其他线程则无法获取。Looper、ActivityThread 以及 AMS 中都用到了 ThreadLocal。当不同线程访问同一个ThreadLocal 的 get方法，ThreadLocal 内部会从各自的线程中取出一个数组，然后再从数组中根据当前 ThreadLcoal 的索引去查找对应的value值。
``ThreadLocal.java``
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

···
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
```

### MessageQueue
MessageQueue主要包含两个操作：插入和读取。读取操作本身会伴随着删除操作，插入和读取对应的方法分别是 ``enqueueMessage`` 和 ``next``。MessageQueue 内部实现并不是用的队列，实际上通过一个单链表的数据结构来维护消息列表。next 方法是一个无限循环的方法，如果消息队列中没有消息，那么 next 方法会一直阻塞。当有新消息到来时，next 方法会放回这条消息并将其从单链表中移除。

``MessageQueue.java``
```java
boolean enqueueMessage(Message msg, long when) {
    ···
    synchronized (this) {
        ···
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
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
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
···
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    ···
    for (;;) {
        ···
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
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
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
            ···
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

### Looper
Looper 会不停地从 MessageQueue 中 查看是否有新消息，如果有新消息就会立刻处理，否则会一直阻塞。
``Looper.java``
```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

可通过 Looper.prepare() 为当前线程创建一个 Looper：
```java
new Thread("Thread#2") {
    @Override
    public void run() {
        Looper.prepare();
        Handler handler = new Handler();
        Looper.loop();
    }
}.start();
```

除了 prepare 方法外，Looper 还提供了 ``prepareMainLooper`` 方法，主要是给 ActivityThread 创建 Looper 使用，本质也是通过 prepare 方法实现的。由于主线程的 Looper 比较特殊，所以 Looper 提供了一个 getMainLooper 方法来获取主线程的 Looper。

Looper 提供了 ``quit`` 和 ``quitSafely`` 来退出一个 Looper，二者的区别是：``quit`` 会直接退出 Looper，而 ``quitSafly`` 只是设定一个退出标记，然后把消息队列中的已有消息处理完毕后才安全地退出。Looper 退出后，通过 Handler 发送的消息会失败，这个时候 Handler 的 send 方法会返回 false。因此在不需要的时候应终止 Looper。

``Looper.java``
```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    ···
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        ···
        try {
            msg.target.dispatchMessage(msg);
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        ···
        msg.recycleUnchecked();
    }
}
```
loop 方法是一个死循环，唯一跳出循环的方式是 MessageQueue 的 next 方法返回了null。当 Looper 的 quit 方法被调用时，Looper就会调用 MessageQueue 的 quit 或者 qutiSafely 方法来通知消息队列退出，当消息队列被标记为退出状态时，它的 next 方法就会返回 null。loop 方法会调用 MessageQueue 的 next 方法来获取新消息，而 next 是一个阻塞操作，当没有消息时，next 会一直阻塞，导致 loop 方法一直阻塞。Looper 处理这条消息： msg.target.dispatchMessage(msg)，这里的 msg.target 是发送这条消息的 Handler 对象。

### Handler
Handler 的工作主要包含消息的发送和接收的过程。消息的发送可以通过 post/send 的一系列方法实现，post 最终也是通过send来实现的。

![](https://img-blog.csdnimg.cn/20181220142659447)

# 线程异步
线程（thread） 是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。

应用启动时，系统会为应用创建一个名为“主线程”的执行线程( UI 线程)。 此线程非常重要，因为它负责将事件分派给相应的用户界面小部件，其中包括绘图事件。 此外，它也是应用与 Android UI 工具包组件（来自 ``android.widget`` 和 ``android.view`` 软件包的组件）进行交互的线程。

系统不会为每个组件实例创建单独的线程。运行于同一进程的所有组件均在 UI 线程中实例化，并且对每个组件的系统调用均由该线程进行分派。 因此，响应系统回调的方法（例如，报告用户操作的 onKeyDown() 或生命周期回调方法）始终在进程的 UI 线程中运行。

Android 的单线程模式必须遵守两条规则:
- 不要阻塞 UI 线程
- 不要在 UI 线程之外访问 Android UI 工具包

为解决此问题，Android 提供了几种途径来从其他线程访问 UI 线程:
- ``Activity.runOnUiThread(Runnable)``
- ``View.post(Runnable)``
- ``View.postDelayed(Runnable, long)``

## AsyncTask
AsyncTask 封装了 Thread 和 Handler，并不适合特别耗时的后台任务，对于特别耗时的任务来说，建议使用线程池。

### 基本使用
| 方法 | 说明
|--|--
| onPreExecute() | 异步任务执行前调用，用于做一些准备工作
| doInBackground(Params...params) | 用于执行异步任务，此方法中可以通过 publishProgress 方法来更新任务的进度，publishProgress 会调用 onProgressUpdate 方法
| onProgressUpdate | 在主线程中执行，后台任务的执行进度发生改变时调用
| onPostExecute | 在主线程中执行，在异步任务执行之后

```java
import android.os.AsyncTask;

public class DownloadTask extends AsyncTask<String, Integer, Boolean> {

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
    }

    @Override
    protected Boolean doInBackground(String... strings) {
        return null;
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        super.onProgressUpdate(values);
    }

    @Override
    protected void onPostExecute(Boolean aBoolean) {
        super.onPostExecute(aBoolean);
    }
}
```

- 异步任务的实例必须在 UI 线程中创建，即 AsyncTask 对象必须在UI线程中创建。
- execute(Params... params)方法必须在UI线程中调用。
- 不要手动调用 onPreExecute()，doInBackground()，onProgressUpdate()，onPostExecute() 这几个方法。
- 不能在 doInBackground() 中更改UI组件的信息。
- 一个任务实例只能执行一次，如果执行第二次将会抛出异常。
- execute() 方法会让同一个进程中的 AsyncTask 串行执行，如果需要并行，可以调用 executeOnExcutor 方法。

### 工作原理
``AsyncTask.java``
```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
sDefaultExecutor 是一个串行的线程池，一个进程中的所有的 AsyncTask 全部在该线程池中执行。AysncTask 中有两个线程池（SerialExecutor 和 THREAD_POOL_EXECUTOR）和一个 Handler（InternalHandler），其中线程池 SerialExecutor 用于任务的排队，THREAD_POOL_EXECUTOR 用于真正地执行任务，InternalHandler 用于将执行环境从线程池切换到主线程。

``AsyncTask.java``
```java
private static Handler getMainHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}

private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}


private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

## HandlerThread
HandlerThread 集成了 Thread，却和普通的 Thread 有显著的不同。普通的 Thread 主要用于在 run 方法中执行一个耗时任务，而 HandlerThread 在内部创建了消息队列，外界需要通过 Handler 的消息方式通知 HanderThread 执行一个具体的任务。

``HandlerThread.java``
```java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```

## IntentService
IntentService 可用于执行后台耗时的任务，当任务执行后会自动停止，由于其是 Service 的原因，它的优先级比单纯的线程要高，所以 IntentService 适合执行一些高优先级的后台任务。在实现上，IntentService 封装了 HandlerThread 和 Handler。

``IntentService.java``
```java
@Override
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.

    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```

IntentService 第一次启动时，会在 onCreatea 方法中创建一个 HandlerThread，然后使用的 Looper 来构造一个 Handler 对象 mServiceHandler，这样通过 mServiceHandler 发送的消息最终都会在 HandlerThread 中执行。每次启动 IntentService，它的 onStartCommand 方法就会调用一次，onStartCommand 中处理每个后台任务的 Intent，onStartCommand 调用了 onStart 方法：

``IntentService.java``
```java
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

···

@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```

可以看出，IntentService 仅仅是通过 mServiceHandler 发送了一个消息，这个消息会在 HandlerThread 中被处理。mServiceHandler 收到消息后，会将 Intent 对象传递给 onHandlerIntent 方法中处理，执行结束后，通过 stopSelf(int startId) 来尝试停止服务。（stopSelf() 会立即停止服务，而 stopSelf(int startId) 则会等待所有的消息都处理完毕后才终止服务）。

## 线程池
线程池的优点有以下：
- 重用线程池中的线程，避免因为线程的创建和销毁带来性能开销。
- 能有效控制线程池的最大并发数，避免大量的线程之间因互相抢占系统资源而导致的阻塞现象。
- 能够对线程进行管理，并提供定时执行以及定间隔循环执行等功能。

java 中，ThreadPoolExecutor 是线程池的真正实现：

``ThreadPoolExecutor.java``
```java
/**
    * Creates a new {@code ThreadPoolExecutor} with the given initial
    * parameters.
    *
    * @param corePoolSize 核心线程数
    * @param maximumPoolSize 最大线程数
    * @param keepAliveTime 非核心线程闲置的超时时长
    * @param unit 用于指定 keepAliveTime 参数的时间单位
    * @param 任务队列，通过线程池的 execute 方法提交的 Runnable 对象会存储在这个参数中
    * @param threadFactory 线程工厂，用于创建新线程
    * @param handler 任务队列已满或者是无法成功执行任务时调用
    */
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler) {
    ···
}
```

| 类型 | 创建方法 | 说明
|--|--|--
| FixedThreadPool | Executors.newFixedThreadPool(int nThreads) | 一种线程数量固定的线程池，只有核心线程并且不会被回收，没有超时机制
| CachedThreadPool | Executors.newCachedThreadPool() | 一种线程数量不定的线程池，只有非核心线程，当线程都处于活动状态时，会创建新线程来处理新任务，否则会利用空闲的线程，超时时长为60s
| ScheduledThreadPool | Executors.newScheduledThreadPool(int corePoolSize) | 核心线程数是固定的，非核心线程数没有限制，非核心线程闲置时立刻回收，主要用于执行定时任务和固定周期的重复任务
| SingleThreadExecutor | Executors.newSingleThreadExecutor() | 只有一个核心线程，确保所有任务在同一线程中按顺序执行

# RecyclerView 优化
- 数据处理和视图加载分离：数据的处理逻辑尽可能放在异步处理，onBindViewHolder 方法中只处理数据填充到视图中。

- 数据优化：分页拉取远端数据，对拉取下来的远端数据进行缓存，提升二次加载速度；对于新增或者删除数据通过 DiffUtil 来进行局部刷新数据，而不是一味地全局刷新数据。

示例
```java
public class AdapterDiffCallback extends DiffUtil.Callback {

    private List<String> mOldList;
    private List<String> mNewList;

    public AdapterDiffCallback(List<String> oldList, List<String> newList) {
        mOldList = oldList;
        mNewList = newList;
        DiffUtil.DiffResult
    }

    @Override
    public int getOldListSize() {
        return mOldList.size();
    }

    @Override
    public int getNewListSize() {
        return mNewList.size();
    }

    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        return mOldList.get(oldItemPosition).getClass().equals(mNewList.get(newItemPosition).getClass());
    }

    @Override
    public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
        return mOldList.get(oldItemPosition).equals(mNewList.get(newItemPosition));
    }
}
```
```java
DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(new AdapterDiffCallback(oldList, newList));
diffResult.dispatchUpdatesTo(mAdapter);
```

- 布局优化：减少布局层级，简化 ItemView

- 升级 RecycleView 版本到 25.1.0 及以上使用 Prefetch 功能

- 通过重写 RecyclerView.onViewRecycled(holder) 来回收资源

- 如果 Item 高度是固定的话，可以使用 RecyclerView.setHasFixedSize(true); 来避免 requestLayout 浪费资源

- 对 ItemView 设置监听器，不要对每个 Item 都调用 addXxListener，应该大家公用一个 XxListener，根据 ID 来进行不同的操作，优化了对象的频繁创建带来的资源消耗

- 如果多个 RecycledView 的 Adapter 是一样的，比如嵌套的 RecyclerView 中存在一样的 Adapter，可以通过设置 RecyclerView.setRecycledViewPool(pool)，来共用一个 RecycledViewPool。
