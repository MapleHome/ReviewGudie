
从 Android 12（API 级别 31）开始，仅当您的应用获准处理某个通用网络 intent 中包含的特定网域时，该网络 intent 才会解析为应用中的 activity。如果您的应用未获准处理相应的网域，则该网络 intent 会解析为用户的默认浏览器应用。

应用可通过执行以下某项操作来获准处理相应的网域：
- 使用 [Android App Links](https://developer.android.google.cn/training/app-links/verify-site-associations) 验证网域。
- 在以 Android 12 或更高版本为目标平台的应用中，系统更改了其[自动验证](https://developer.android.google.cn/training/app-links/verify-site-associations#auto-verification)应用的 Android App Links 的方式。在应用的 [intent 过滤器](https://developer.android.google.cn/training/app-links/verify-site-associations#add-intent-filters)中，检查是否包含 BROWSABLE 类别并支持 https 方案。
在 Android 12 或更高版本中，您可以[手动验证](https://developer.android.google.cn/training/app-links/verify-site-associations#manual-verification)应用的 Android App Links，来测试此更新后的逻辑将如何影响您的应用。
- 在系统设置中[请求用户将您的应用与网域相关联](https://developer.android.google.cn/training/app-links/verify-site-associations#request-user-associate-app-with-domain)。

如果您的应用调用网络 intent，不妨考虑添加一个提示或对话框，要求用户确认操作。


---

> 网络 intent 解析

微博Android App 内部分 activity 存在 http/https 数据架构的 intent 过滤器，根据 Android 12 系统要求，需要在对应网址下有相应的验证文件并验证通过，才能被对应activity响应跳转到该页面，否则跳转到系统默认浏览器。

经整理 微博app 内共有
`<data android:scheme="http" android:host="weibo.cn" android:path="/qr/xxxx" />` // 77 处
`<data android:scheme="http" android:host="wbpay" />` // 1处

需要在 https://weibo.cn/.well-known/assetlinks.json 和 https://wbpay/.well-known/assetlinks.json 位置添加 assetlinks.json 文件。
内容为：
```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target" : { "namespace": "android_app", "package_name": "com.sina.weibo",
               "sha256_cert_fingerprints": ["A0:78:C3:0F:0A:58:E6:3D:06:CA:5F:C7:C4:21:95:F6:A4:94:02:15:D4:BC:D2:3D:94:0B:7E:A0:3E:86:37:9D"] }
}]
```

需要保证：
可通过 HTTPS 连接访问 assetlinks.json 文件。 assetlinks.json 文件必须无需重定向即可访问（无 301 或 302 重定向），且可由漫游器访问（您的 robots.txt 必须允许抓取 /.well-known/assetlinks.json）。

测试网址：
https://developers.google.com/digital-asset-links/tools/generator


#### 检查

获取设备上所有应用的现有链接处理政策列表：
`adb shell dumpsys package domain-preferred-apps` 或
`adb shell dumpsys package d`

```java
shaoshuai@shaoshuaideMacBook-Pro ~ % adb shell dumpsys package d

App verification status:
  // 未通过
  Package: com.homelink.android
  Domains: linkm.lianjia.com
  Status:  ask

  // 验证通过
  Package: com.maple.ishare
  Domains: www.nb2021.top
  Status:  always : 200000000

  Package: com.sina.weibo
  Domains: weibo.cn wbpay cps.business.taobao.com
  Status:  ask
```

```
adb shell am start-activity -d 'http://weibo.cn/qr/message'
adb shell am start-activity -d 'http://www.nb2021.top'
adb shell am start-activity -d 'http://www.nb2021.top/language'
adb shell am start -a android.intent.action.VIEW \
        -c android.intent.category.BROWSABLE \
        -d "http://www.nb2021.top"
```

- 华为鸿蒙2.0
  当一个app验证通过后，直接跳app对应页面

- onePlus 12
  SecurityException: Permission Denial: 异常，无响应






----
