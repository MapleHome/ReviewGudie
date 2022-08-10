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
具体介绍可以看看作者的说明：http://blog.zhaiyifan.cn/2016/01/16/BlockCanaryTransparentPerformanceMonitor/
