Android 12 适配 - PendingIntent 可变性 适配

> PendingIntent

Android12新特性要求应用创建的每个PendingIntent对象都要指定可变性, 使用PendingIntent.FLAG_MUTABLE 或PendingIntent.FLAG_IMMUTABLE标志。
```java
PendingIntent pendingIntent = PendingIntent.getActivity(
getApplicationContext(), REQUEST_CODE, intent, PendingIntent.FLAG_IMMUTABLE);
```
> 检查项目中的使用情况

1. 要查的方法放到 `res/permission.method` 里面，替换掉文件里现有的：
android/app/PendingIntent.getActivity
android/app/PendingIntent.getBroadcast
android/app/PendingIntent.getService

2. 在 Terminal 中执行：
./gradlew  clean
./gradlew  app:assembleDebug
./gradlew  app:methodFindForDebug

---
- 未适配

肖肖
扫描调用 : com/vivo/push/util/NotifyAdapterUtil -> android/app/PendingIntent.getService // 268435456
扫描调用 : com/meizu/cloud/pushsdk/b/a/a -> android/app/PendingIntent.getBroadcast // 1073741824
扫描调用 : com/meizu/cloud/pushsdk/handler/a/c -> android/app/PendingIntent.getService // 67108864
扫描调用 : com/meizu/cloud/pushsdk/notification/a -> android/app/PendingIntent.getBroadcast // 1073741824

宋鹏
扫描调用 : com/sina/wbim/net/manager/WBIMPushAlarmManager -> android/app/PendingIntent.getBroadcast // 0

钰进
扫描调用 : com/taobao/taolive/room/ui/logo/LogoController -> android/app/PendingIntent.getBroadcast // 134217728


扫描调用 : androidx/appcompat/widget/SearchView -> android/app/PendingIntent.getActivity
















---
