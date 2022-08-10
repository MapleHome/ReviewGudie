### Android 11 升级适配

res/script/config_build.gradle
```java
    android_build = [
            compileSdkVersion                   : 29,
            buildToolsVersion                   : '29.0.3',
            minSdkVersion                       : 21,
            targetSdkVersion                    : 29
    ]
    android_build = [
            compileSdkVersion                   : 30,
            buildToolsVersion                   : '30.0.1',
            minSdkVersion                       : 21,
            targetSdkVersion                    : 30
    ]
```
config.gradle
```java
    TARGET_SDK_VERSION=29
    COMPILE_SDK_VERSION=29
    BUILD_TOOLS_VERSION='29.0.3'

    TARGET_SDK_VERSION=30
    COMPILE_SDK_VERSION=30
    BUILD_TOOLS_VERSION='30.0.1'
```




#### 存储适配
Scoped Storage ：分区存储、作用域存储
Android长久以来的支持外置存储功能（SD卡存储）。

好处：
- 虚报占用空间。
  存储在SD卡的文件不会计入到应用程序的占用空间当中，也就是说即使你在SD卡存放了1G的文件，你的应用程序在设置中显示的占用空间仍然可能只有几十K。
- 数据永久存储。
  存储在SD卡的文件，即使应用程序被卸载了，这些文件仍然会被保留下来，有助于实现数据永久保留。

坏处：
- app随意创建自己的目录弄的SD卡乱糟糟的。
- 卸载了app后，缓存的垃圾文件依旧保留在手机上。
- 安全性问题，SD卡所有程序都有权随意访问。

> 在Android 10 即 `targetSdkVersion = 29` 中，开始推行分区存储。

- 私有目录：每个应用程序只能有权在自己的外置存储空间关联目录下读取和创建文件，获取该关联目录的代码是：`context.getExternalFilesDir()`。关联目录对应的路径: `/storage/emulated/0/Android/data/<包名>/files`
- 公有目录：访问媒体库查看添加图片、音频、视频。通过 `MediaStore` API 来进行访问，`/storage/emulated/0/Downloads`

**适配方案**：
  1. 如果 targetSdkVersion 低于 29，可以不做任何适配。
  2. 如果 targetSdkVersion = 29
      - 可配置 `android:requestLegacyExternalStorage="true"` 暂不启动分区存储。（权宜之计）
      - 严格遵循规范分区存储。在自己的专属目录存储。

**现状**：
  - 项目中，开启了`requestLegacyExternalStorage`配置，并启动分区存储。所以线上不会有任何问题。
  - 但是项目中某些存储逻辑，依旧在公共区进行存储。（技术债务）


> 在Android 11 即 `targetSdkVersion = 30` 的适配中，强制开启分区存储。

**应对方案**：
  1. 申请对整个SD卡的读写权限 `MANAGE_EXTERNAL_STORAGE`，适用于文件管理器类App，不推荐。
  2. 对于app升级覆盖安装用户,可以增加 `android:preserveLegacyExternalStorage="true"`，暂时关闭分区存储，好让开发者完成数据迁移的工作.
  3. 对于卸载安装用户，强制开启分区存储。此时需要规范化全部存储逻辑，处理之前的技术债务。
      - 排查项目中所有在SD卡公共目录存储的地方，规范化存储。


参考资料:
[拖不得了，Android11真的来了，最全适配实践指南奉上](https://mp.weixin.qq.com/s/ihn0uijiNBGRoJX_gpvTrA)
[Android 开发者: 访问应用专属文件](https://developer.android.google.cn/training/data-storage/app-specific#external)
[Android 10适配要点，作用域存储](https://blog.csdn.net/guolin_blog/article/details/105419420/)
[Android 11新特性，Scoped Storage又有了新花样](https://guolin.blog.csdn.net/article/details/113954552)


#### 软件包可见性
当应用查询设备上已安装应用的列表时，系统会过滤返回的列表。
Android 11 更改了应用查询用户已在设备上安装的其他应用以及与之交互的方式。使用新的 元素，应用可以定义一组自身可访问的其他应用。

`packageManager.getInstalledApplications(PackageManager.GET_META_DATA)`，
在Android11版本，只能查询到自己应用和系统应用的信息，查不到其他应用的信息了

Android11中添加 `QUERY_ALL_PACKAGES` 权限,可以获取全部包信息，但是会被严格审核。


#### 电话号码权限
TelecomManager 类中的 `getLine1Number()` 和 `getMsisdn()` 方法.
原来的`READ_PHONE_STATE`权限不管用了，需要`READ_PHONE_NUMBERS`权限才行。


#### Toast适配：
从 Android 11 开始，已弃用自定义消息框视图。
不允许自定义toast从后台显示，会有警告信息，不会崩溃。
```
W/NotificationService: Blocking custom toast from package com.example.studynote due to package not in the foreground
```
⚠️ ：不同厂商机型适配。
在小米10的Android 11 中，可能有闪退问题。参考：https://www.wangt.cc/2021/01/android-11%E9%80%82%E9%85%8D%E6%8C%87%E5%8D%97%E4%B9%8Btoast%E8%A7%A3%E6%9E%90/


#### webView加载本地图片
target 30 中：使用 webView.loadDataWithBaseURL 加载本地图片失败，导致UI显示错误的现象。

需要开启 webView.getSettings().setAllowFileAccess(true)配置。



#### 过时方法：
**AsyncTask** 在Android 11已经不建议使用，建议迁移至kotlin的协程
**Handler** 未指定Looper的构造方法也已不建议使用。
```java
// 建议明确指定Looper：
private Handler handler = new Handler(Looper.myLooper());
private Handler handler = new Handler(Looper.getMainLooper());
```


#### 后台位置信息访问权限
从Android10系统的设备开始，就需要请求后台位置权限`ACCESS_BACKGROUND_LOCATION`，并选择Allow all the time （始终允许）才能获得后台位置权限。
Android11设备上再次加强对后台权限的管理，主要表现在系统对话框上，对话框不再提示始终允许字样，而是提供了位置权限的设置入口，需要在设置页面选择始终允许才能获得后台位置权限。

⚠️ ：`targetSdkVersion = 30`的时候，必须单独申请后台位置权限，而且要在获取前台权限之后，顺序不能乱。并且无任何提示，需要开发者自己设计提示样式。

先申请前台位置权限：`Manifest.permission.ACCESS_COARSE_LOCATION`
再申请后台位置权限：`Manifest.permission.ACCESS_BACKGROUND_LOCATION`


#### 5G
Android 11 添加了在您的应用中支持 5G 的功能
新的Android11也是支持了5G相关的一些功能，包括： 检测是否连接到了5G网络 检查按流量计费性


#### APK 签名方案 v2
对于以 Android 11（API 级别 30）为目标平台，且目前仅使用 APK 签名方案 v1 签名的应用，现在还必须使用 APK 签名方案 v2 或更高版本进行签名。用户无法在搭载 Android 11 的设备上安装或更新仅通过 APK 签名方案 v1 签名的应用。


#### 前台服务类型
Android 10中，在前台服务访问位置信息，需要在对应的service中添加 location 服务类型。
Android 11中，在前台服务访问摄像头或麦克风，需要在对应的service中添加camera或microphone 服务类型。
```java
<manifest>
    ...
   <service
       android:name="MyService"
       android:foregroundServiceType="microphone|camera" />
</manifest>
```
这一限制的变更，使得程序无法在后台启动服务访问摄像头和麦克风。如需使用，只能是前台开启前台服务。


#### 其他变更：
文档访问限制
限制对 APN 数据库的读取访问
在元数据文件中声明“无障碍”按钮使用情况
自动重置权限：如果用户几个月未与应用互动，系统会自动重置应用的敏感权限。
Firebase JobDispatcher 和 GCMNetworkManager
设备到设备文件传输


#### GetNetworkType in Android 11
https://stackoverflow.com/questions/62692649/getnetworktype-in-android-11


#### NetworkInterface
.getHardwareAddress()

----
