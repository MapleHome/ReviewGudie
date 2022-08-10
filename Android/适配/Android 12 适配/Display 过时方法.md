
背景：
Android 设备有许多不同的外形规格，如大屏设备、平板电脑和可折叠设备。为让每种设备都能适当地呈现内容，应用需要正确的获取屏幕或显示屏大小。

随着时间的推移，Android 提供了不同的 API 来获取这些信息。
在 Android 11 中，Google 引入了 WindowMetrics API 并废弃了以下方法：
- Display.getSize()
- Display.getMetrics()

在 Android 12 中，继续建议使用 WindowMetrics，并且正在逐步废弃以下方法：
- Display.getRealSize()
- Display.getRealMetrics()

现状：
微博中存在大量的获取屏幕宽高的方法，但是都没有兼容Android 11 12。
现在进行方法升级适配，提供兼容全版本的工具类方法，对项目内的方法进行替换。
Android 11版本之前，仍使用旧方法获取显示宽高和屏幕宽高。
Android 11、12 使用系统新方法获取app显示宽高和设备屏幕宽高。

功能开关：display_size_use_old 反向开关，默认：未添加AB，功能开启
旧方法对比开关：display_old_method_contrast 默认：未添加AB，功能关闭

测试点：
因为是基础方法修改，无法给出具体的测试用例，常规功能测试即可。
在 Android 11 or 12 的手机上运行app，页面UI 显示正常。

辅助测试项：工程模式 打开 Logutil开关
通过筛选 logTag: Display12 可查看代码中获取宽高的具体值。
添加AB: display_old_method_contrast 可查看新旧方法对比值。

本次测试值更改了 res、sdk 模块公共方法，具体各业务线模块内代码适配工作，各业务线单独提测。

测试包：
http://ota.client.weibo.cn/static/android/com.sina.weibo/com.sina.weibo_weibo_display_12_2022.01.20.14.24.48.apk


辛苦测试～




为了缓解应用使用 Display API 检索应用边界的行为，Android 12 限制了 API 为不完全可调整大小的应用返回的值。这可能会对将此信息与 MediaProjection 一起使用的应用产生影响。

应用应使用 WindowMetrics API 查询其窗口的边界，并使用 Configuration.densityDpi 查询当前的密度。

为了与较低的 Android 版本实现更广泛的兼容性，您可以使用 Jetpack WindowManager 库，它包含一个 WindowMetrics 类，该类支持 Android 4.0（API 级别 14）及更高版本。





----
