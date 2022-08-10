### Service与子线程

关于Service，我的第一反应是运行在后台的服务。

关于后台，我的第一反应又是子线程。

那么Service和子线程到底是什么关系呢？

Service有两个比较重要的元素：

- 长时间运行。Service可以在Activity被销毁，程序被关闭之后都可以继续运行。
- 不提供界面的应用组件。这其实解释了后台的意义，Service的后台指的是不和界面交互，不依赖UI元素。

而且比较关键的点是，**Service也是运行在主线程之中**。

所以运行在后台的Service和运行在后台的线程区别还是挺大的。

- 首先，所运行的线程不同。Service还是运行在主线程，而子线程肯定是开辟了新的线程。
- 其次，后台的概念不同。Service的后台指的是不与界面交互，子线程的后台指的是异步运行。
- 最后，Service作为四大组件之一，控制它也更方便，只要有上下文就可以对其进行控制。

当然，虽然两者概念不同，但是还是有很多合作之处。
Service作为后台运行的组件，其实很多时候也会被用来做耗时操作，那运行在主线程的Service肯定不能直接进行耗时操作，这就需要子线程了。
开启一个后台Service，然后在Service里面进行子线程操作，这样的结合给项目带来的可能性就更大了。
Google也是考虑到这一点，设计出了IntentService这种已经结合好的组件供我们使用。

### IntentService

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
}
```    
理一下这个源码：
首先，这是一个Service
并且内部维护了一个HandlerThread，也就是有完整的Looper在运行。
还维护了一个子线程的ServiceHandler。
启动Service后，会通过Handler执行onHandleIntent方法。
完成任务后，会自动执行stopSelf停止当前Service。
所以，这就是一个可以在子线程进行耗时任务，并且在任务执行后自动停止的Service。


### 后台和前台Service

这就涉及到Service的分类了。

如果从是否无感知来分类，Service可以分为前台和后台。前台Service会通过通知的方式让用户感知到，后台有这么一个玩意在运行。

比如音乐类APP，在后台播放音乐的同时，可以发现始终有一个通知显示在前台，让用户知道，后台有一个这么音乐相关的服务。

在Android8.0，Google要求如果程序在后台，那么就不能创建后台服务，已经开启的后台服务会在一定时间后被停止。

所以，建议使用前台Service，它拥有更高的优先级，不易被销毁。使用方法如下：
```java
    startForegroundService(intent);

    public void onCreate() {
        super.onCreate();
        Notification notification = new Notification.Builder(this)
                .setChannelId(CHANNEL_ID)
                .setContentTitle("主服务")//标题
                .setContentText("运行中...")//内容
                .setSmallIcon(R.mipmap.ic_launcher)
                .build();
        startForeground(1,notification);
    }  


    <!--android 9.0上使用前台服务，需要添加权限-->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```
那后台任务该怎么办呢？官方建议使用 JobScheduler 。

### 说说 JobScheduler

任务调度JobScheduler，Android5.0被推出。（可能有的朋友感觉比较陌生，其实他也是通过Service实现的，这个待会再说）

它能做的工作就是可以在你所规定的要求下进行自动任务执行。比如规定时间、网络为WIFI情况、设备空闲、充电时等各种情况下后台自动运行。

所以Google让它来替代后台Service的一部分功能，使用：

首先，创建一个JobService：
```java
public class MyJobService extends JobService {

    @Override
    public boolean onStartJob(JobParameters params) {
        return false;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        return false;
    }
}  
```
然后，注册这个服务（因为JobService也是Service）
```java
<service android:name=".MyJobService"
    android:permission="android.permission.BIND_JOB_SERVICE" />
```
最后，创建一个JobInfo并执行
```java
 JobScheduler scheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);  
 ComponentName jobService = new ComponentName(this, MyJobService.class);

 JobInfo jobInfo = new JobInfo.Builder(ID, jobService)
         .setMinimumLatency(5000)// 任务最少延迟时间
         .setOverrideDeadline(60000)// 任务deadline，当到期没达到指定条件也会开始执行
         .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)// 网络条件，默认值NETWORK_TYPE_NONE
         .setRequiresCharging(true)// 是否充电
         .setRequiresDeviceIdle(false)// 设备是否空闲
         .setPersisted(true) //设备重启后是否继续执行
         .setBackoffCriteria(3000，JobInfo.BACKOFF_POLICY_LINEAR) //设置退避/重试策略
         .build();  
 scheduler.schedule(jobInfo);
```
简单说下原理：

JobSchedulerService是在SystemServer中启动的服务，然后会遍历没有完成的任务，通过Binder找到对应的JobService，执行onStartJob方法，完成任务。具体可以看看参考链接的分析。

所以也就知道了，在5.0之后，如果有需要后台任务执行，特别是需要满足一定条件触发的任务，比如网络电量等等情况，就可以使用JobScheduler。

有的人可能要问了，5.0之前怎么办呢？

- 可以使用GcmNetworkManager或者BroadcastReceiver等处理部分情况下的任务需求。

Google也是考虑到了这一点，所以将5.0之后的JobScheduler和5.0之前的GcmNetworkManager、GcmNetworkManager、AlarmManager等和任务相关的API相结合，设计出了WorkManager。


### 说说 WorkManager

WorkManager 是一个 API，可供您轻松调度那些即使在退出应用或重启设备后仍应运行的可延期异步任务。

作为Jetpack的一员，并不算很新的内容，它的本质就是结合已有的任务调度相关的API，然后根据版本需求等来执行这些任务，官网有一张图：

![](https://mmbiz.qpic.cn/mmbiz_png/jfNJsrA21FLsoR4wkpEVHEMLpFSMp3MyW91wwzlu6d6cQx7gqQJWTfvdYRvyL4p1NH4KObOTpnquef6GjpnGNA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

所以WorkManager到底能做什么呢？

1、对于一些任务约束能很好的执行，比如网络、设备空闲状态、足够存储空间等条件下需要执行的任务。
2、可以重复、一次性、稳定的执行任务。包括在设备重启之后都能继续任务。
3、可以定义不同工作任务的衔接关系。比如设定一个任务接着一个任务。

总之，它是后台执行任务的一大利器。
