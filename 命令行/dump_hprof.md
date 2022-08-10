


要查的方法放到res/permission.method 里面，替换掉文件里现有的
./gradlew  clean
./gradlew  app:assembleDebug
./gradlew  app:methodFindForDebug


 isShowSystemWarning  false - 0
 isShowSystemWarning  true - 1
 isShowSystemWarning  false - 2
 isShowSystemWarning  false - 2


钰进（已适配，阿里百川sdk初始化不成功，才会调用，正常逻辑走不到～）
扫描调用 : com/sina/weibo/wblive/taobao/widgets/utils/CommonUtils -> android/content/pm/PackageManager.getInstalledPackages  
扫描调用 : com/sina/weibo/wblive/taobao/widgets/utils/TBLivePopControl -> android/content/pm/PackageManager.getInstalledPackages

艳艳（相应代码已删除）
扫描调用 : com/sina/weibo/appmarket/sng/utils/CommonUtils -> android/content/pm/PackageManager.getInstalledPackages  
扫描调用 : com/sina/weibo/appmarket/v3/widget/GameDetailInfoRelativeLayout -> android/content/pm/PackageManager.getInstalledPackages

玉刚 -> 杨帅（已适配）
扫描调用 : com/sina/weibo/utils/Utils -> android/content/pm/PackageManager.getInstalledPackages  

alibc_link_partner:4.1.27@aar 玉刚
扫描调用 : com/alibaba/alibclinkpartner/smartlink/manager/ALSLAppCheckManager -> android/content/pm/PackageManager.getInstalledPackages

publisher:1.0.21-14-4@aar  佳亮（可为空，不适配）
扫描调用 : com/tencent/mars/comm/NetStatusUtil -> android/content/pm/PackageManager.getInstalledPackages

wbplugin:1.0.3  志辉
扫描调用 : com/sina/weibo/wbplugin/internal/PluginPackageManager -> android/content/pm/PackageManager.getInstalledPackages
扫描调用 : com/sina/weibo/wbplugin/internal/PluginPackageManager -> android/content/pm/PackageManager.getInstalledApplications

wbssdkexport:0.0.33 志辉
扫描调用 : com/sina/wbs/load/tasks/SearchApkFromProviderTask -> android/content/pm/PackageManager.getInstalledPackages

taoliveroom:1.3.4.5-wb@aar
扫描调用 : com/taobao/taolive/room/utils/AndroidUtils -> android/content/pm/PackageManager.getInstalledPackages

alipaysdk:1.0@aar （搜到，实际没有）
扫描调用 : com/alipay/security/mobile/module/deviceinfo/a -> android/content/pm/PackageManager.getInstalledPackages







---

jhat /Users/shaoshuai/Downloads/dump/test2.hprof



微博 target 31 升级 步骤：
1. 架构组 合并 功能测试。粗力度确认：常规功能正常
2. 所有业务线 深度测试 确认。细粒度确认：细节功能正常
3. 上线 google 渠道。线上实测
4. 上线 主版本。



---


java -jar /Users/shaoshuai/Downloads/dump/DumpPrinter-1.0.jar /Users/shaoshuai/Downloads/dump/test.hprof > dump_log.txt

hprof-conv /Users/shaoshuai/Downloads/test.hprof  /Users/shaoshuai/Downloads/test2.hprof


adb shell ps | grep com.sina.weibo

- 分析本地文件
/Users/work/Android/shark-cli-2.9.1/bin/shark-cli --hprof /Users/shaoshuai/Downloads/dump/test.hprof analyze

- 分析正在运行的app 获取 分析文件
/Users/work/Android/shark-cli-2.9.1/bin/shark-cli --process com.sina.weibo analyze > wb_analyze_10C6095010_xxx.text


1. 下载 shark-cli
下载地址：https://github.com/square/leakcanary/releases/download/v2.9.1/shark-cli-2.9.1.zip

2. 将 BaseTestRunner 中的
WeiboMatrixConfigs.checkUploadIssues(); 替换成：
WeiboMatrixConfigs.testCaseFinish();

执行玩测试流程后，会生成 /storage/emulated/0/Android/data/com.sina.weibo/files/hprof/wb_test_case.hprof 文件。

3. 从手机拉取hprof文件到PC
adb pull /storage/emulated/0/Android/data/com.sina.weibo/files/hprof/wb_test_case.hprof /PC目录/aabbcc.hprof

需要判断本地是否有hprof文件

4. 分析本地 hprof 文件 生成分析报告
/Users/work/Android/shark-cli-2.9.1/bin/shark-cli --hprof /PC目录/aabbcc.hprof analyze > wb_analyze_10C5195010_xxx.text

5. 将分析报告文件推送到手机
adb push wb_analyze_10C5295010_123.text /storage/emulated/0/Android/data/com.weibo.wbmatrix/files/hprof

adb push wb_analyze_10C6195010_x1.text /storage/emulated/0/Android/data/com.weibo.wbmatrix/files/hprof



6. 收尾，删除产生的hprof文件和分析文件
