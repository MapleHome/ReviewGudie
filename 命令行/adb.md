## adb命令

- `adb devices` 查看所有设备
- `adb shell getprop` 可以查看手机上所有属性值。
- `abb shell getprop xxx` 可以查看某个指定属性值。
- `adb logcat |grep FixCrash` 查看指定tag日志

安装卸载app
- `adb install /Users/Downloads/app-debug.apk` 安装app到手机
- `adb uninstall com.sina.weibo` 卸载手机上的应用

测试
- `adb shell am instrument -w -r -e debug false -e class com.sina.weibo.testcase.DBTestCase#testCheckWeiboDBVersion_1200101 com.sina.weibo.test/com.sina.weibo.testsdk.NewInstrumentationTestRunner` 执行测试用例
