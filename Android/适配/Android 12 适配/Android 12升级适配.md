Android 12 适配
https://developer.android.google.cn/about/versions/12/deprecations

 Android 12 的已知问题，请参阅
 https://developer.android.com/about/versions/12/top-issues

OPPO Android 12 应用兼容性适配指导
https://open.oppomobile.com/wiki/doc#id=10960

XiaoMi Android 12 应用适配指南
https://dev.mi.com/console/doc/detail?pId=2439

---

#### 通配性问题

> 应用启动画面

如果您之前在 Android 11 或更低版本中实现了自定义启动画面，则需要将您的应用迁移到 SplashScreen API，以确保它从 Android 12 开始正确显示。如果不迁移您的应用，则可能会导致应用启动体验变差或出乎预期。

如需了解相关说明，请参阅 [将现有的启动画面实现迁移到 Android 12](https://developer.android.google.cn/guide/topics/ui/splash-screen/migrate)。

此外，从 Android 12 开始，在所有应用的[冷启动](https://developer.android.google.cn/topic/performance/vitals/launch-time#cold)和[温启动](https://developer.android.google.cn/topic/performance/vitals/launch-time#warm)期间，系统始终会应用新的 [Android 系统默认启动画面](https://developer.android.google.cn/about/versions/12/features/splash-screen)。 默认情况下，此系统默认启动画面由应用的启动器图标元素和主题的 windowBackground（如果是单色）构成。

如需了解详情，请参阅[启动画面开发者指南](https://developer.android.google.cn/guide/topics/ui/splash-screen)。

> 网络 intent 解析

 详见 `../Android 12 scheme 适配.md`

> 拉伸滚动效果

在搭载 Android 12 及更高版本的设备上，[滚动事件](https://developer.android.google.cn/training/gestures/scroll#term) 的视觉行为发生了变化。

在 Android 11 及更低版本中，滚动事件会使视觉元素发光。在 Android 12 及更高版本中，发生拖动事件时，视觉元素会拉伸和反弹；发生快速滑动事件时，它们会快速滑动和反弹。

如需了解详情，请参阅 [动画演示滚动手势](https://developer.android.google.cn/training/gestures/scroll) 指南。

- OverScroll 滚动动画

Android 12 修改了 OverScroll 的效果动画，从原来的拉到底部显示蓝光，修改成了拉到变形弹弹弹的动画
也可以通过 xml 中设置 android:overScrollMode="never" 或者使用代码设置 recyclerview.setOverScrollMode(View.OVER_SCROLL_NEVER); 来屏蔽默认的滚动动画。

> 沉浸模式下的手势导航改进

Android 12 整合了现有行为，让用户可以在沉浸模式下更轻松地执行手势导航命令。此外，Android 12 还为粘性沉浸模式提供了向后兼容性行为。


> Display 方法废弃

继 Android 11 废弃了 Display.getSize() 和 Display.getMetrics() 后，
Android 12 上进一步废弃了 Display.getRealMetrics() 和 Display.getRealSize()
推荐使用 WindowMetrics, 并且谷歌提供了一个兼容到 Android 4.0 的 WindowManager 兼容库。
```java
if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.R) {
     metrics = activity.getWindowManager().getCurrentWindowMetrics();
     int width = metrics.getBounds().width();
     int height = metrics.getBounds().height();
}
```

> Display#getRealSize 和 getRealMetrics：废弃和限制

Android 设备有许多不同的外形规格，如大屏设备、平板电脑和可折叠设备。为了针对每种设备适当地呈现内容，您的应用需要确定屏幕或显示屏尺寸。随着时间的推移，Android 提供了不同的 API 来检索这些信息。
在 Android 11 中，我们引入了 WindowMetrics API 并废弃了以下方法：
- Display.getSize()
- Display.getMetrics()

在 Android 12 中，我们继续建议使用 WindowMetrics，并且正在逐步废弃以下方法：
- Display.getRealSize()
- Display.getRealMetrics()

为了缓解应用使用 Display API 检索应用边界的行为，Android 12 限制了 API 为不完全可调整大小的应用返回的值。这可能会对将此信息与 MediaProjection 一起使用的应用产生影响。

应用应使用 WindowMetrics API 查询其窗口的边界，并使用 Configuration.densityDpi 查询当前的密度。

为了与较低的 Android 版本实现更广泛的兼容性，您可以使用 Jetpack WindowManager 库，它包含一个 WindowMetrics 类，该类支持 Android 4.0（API 级别 14）及更高版本。


> 移除了 BouncyCastle 实现

Android 12 移除了之前弃用的加密算法的许多 BouncyCastle 实现，包括所有 AES 算法。系统改用这些算法的 Conscrypt 实现。

> 剪贴板访问通知

在 Android 12 及更高版本中，当某个应用首次调用 `getPrimaryClip()` 以从另一个应用访问剪辑数据时，会弹出一个消息框消息，通知用户对剪贴板的访问。

消息框消息内的文本包含以下格式：`APP pasted from your clipboard.`
```
注意：您的应用可能会调用 getPrimaryClipDescription() 以接收有关剪贴板上当前数据的信息。当您的应用调用此方法时，系统不会显示消息框消息。
```
**有关剪辑说明中文本的信息**
在 Android 12 及更高版本中，`getPrimaryClipDescription()` 可以检测到以下详细信息：
- 使用 `isStyledText()` 检测样式化文本。
- 使用 `getConfidenceScore()` 检测文本的不同分类，如网址。

> 引入限制域概念

获取 APP 当前的限制域可以使用 getAppStandbyBucket 方法
```java
if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
    UsageStatsManager manager = (UsageStatsManager) getSystemService(USAGE_STATS_SERVICE);
    if(manager != null) manager.getAppStandbyBucket();
}
```
需要测试 APP 在严格限制域的表现，谷歌也提供了相对应的命令来进行模拟:
adb shell am set-standby-bucket PACKAGE_NAME restricted


> 非可信触摸事件会被屏蔽

Android 12 开始，如果 APP 被其他UI遮挡覆盖或者在APP上绘制了其他UI，用户所有的触摸(touch)事件不再会传递下来了



---

#### 全面适配问题

> WebView 中的现代 SameSite Cookie 行为

Cookie 的 SameSite 属性决定了它是可以与任何请求一起发送，还是只能与同站点请求一起发送。Android 12 中的 WebView 基础版本（版本 89.0.4385.0）包含以下隐私保护方面的变更，旨在改善对第三方 Cookie 的默认处理方式，并帮助防止意外跨站点共享：

没有 SameSite 属性的 Cookie 被视为 SameSite=Lax。
带有 SameSite=None 的 Cookie 还必须指定 Secure 属性，这意味着它们需要安全的上下文，并应通过 HTTPS 发送。
站点的 HTTP 版本和 HTTPS 版本之间的链接现在被视为跨站点请求，因此除非将 Cookie 正确标记为 SameSite=None; Secure，否则 Cookie 不会被发送。

> 更安全的组件导出

如果您对四个组件配置了intent 过滤器，则务必要在代码中显式声明android:exported 属性。如果未设置该属性，那应用将无法安装在 Android12 上。
```xml
<service android:name="com.example.app.backgroundService"
         android:exported="false">
    <intent-filter>
        <action android:name="com.example.app.START_BACKGROUND" />
    </intent-filter>
</service>
```

android:exported
此元素设置 Activity 是否可由其他应用的组件启动 —“true”表示可以，“false”表示不可以。若为“false”，则 Activity 只能由同一应用的组件或使用同一用户 ID 的不同应用启动。
如果您使用的是 Intent 过滤器，则不应将此元素设置为“false”。否则，在应用尝试调用 Activity 时，系统会抛出 ActivityNotFoundException 异常。相反，您不应为其设置 Intent 过滤器，以免其他应用调用 Activity。

如果没有 Intent 过滤器，则此元素的默认值为“false”。如果您将元素设置为“true”，则任何知道其确切类名的应用均可访问 Activity，但在系统尝试匹配隐式 Intent 时，该 Activity 无法解析。

此属性并非是限制 Activity 向其他应用公开的唯一方式。您还可使用权限来限制哪些外部实体能够调用 Activity（请参阅 permission 属性）

> PendingIntent

Android12新特性要求应用创建的每个PendingIntent对象都要指定可变性, 使用PendingIntent.FLAG_MUTABLE 或PendingIntent.FLAG_IMMUTABLE标志。
```java
PendingIntent pendingIntent = PendingIntent.getActivity(
getApplicationContext(), REQUEST_CODE, intent, PendingIntent.FLAG_IMMUTABLE);
```


> 不安全的intent启动

Android12为开发者们提供了一种调试功能，如果应用以不安全的方式启动intent，此功能将会发出警告。具体实现也比较简单，开发者在application中添加以下代码即可：
```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
        // Other StrictMode checks that you've previously added.
        .detectUnsafeIntentLaunch().penaltyLog()
        // Consider also adding penaltyDeath()
        .build());
}
```

> 大致位置选项

Android12引入了“大致位置”选项。当应用需要访问位置权限时，弹窗将会出现“确切位置”和“大致位置”两个选项供用户进行授权：
- 确切位置，通常精确到几米之内。
- 大致（粗略）位置，一般为几百米。
位置信息一共有三种授权方式：仅在使用该应用时允许、仅限这一次和不允许。
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```


应用休眠
受标记的应用如果几个月未被使用，则系统会自动取消其权限、停止各种后台通知，将该应用置于休眠状态，以省电并移除其占有的应用空间
开发者可以调用包含 Intent.ACTION_APPLICATION_DETAILS_SETTINGS intent 操作的intent，向用户发送请求，让其准许应用免于休眠和自动重置权限限制。

ADB备份限制
Android12还更改了adb backup命令的默认行为。对于以Android12为目标平台的应用，当运行adb backup命令时，从设备导出的其他任何系统数据都不会包含应用的数据。


GpsStatus API限制
Android 7.0 以后官方推荐使用GNSS。目标平台是Android 12及以上的应用程序，所用到的GpsStatus API必须要用GnssStates API替换。
适配建议：如果应用的targetSdk版本是针对Android 12及以上，为保证功能的正常，请使用GnssStatus.Callback对GpsStatus.Listener进行替代



---
