### 一、ARouter

路由框架在大型项目中比较常见，特别是在项目中拥有多个 moudle 的时候。为了实现组件化，多个 module 间的通信就不能直接以模块间的引用来实现，此时就需要依赖路由框架来实现模块间的通信和解耦😁:

而 ARouter 就是一个用于帮助 Android App 进行组件化改造的框架，支持模块间的路由、通信、解耦

首先借用阿里云栖社区的一段话：我们所使用的原生路由方案一般是通过显式intent和隐式intent两种方式实现的（这里主要是指跳转Activity or Fragment）。在显式intent的情况下，因为会存在直接的类依赖的问题，导致耦合非常严重；而在隐式intent情况下，则会出现规则集中式管理，导致协作变得非常困难。一般而言配置规则都是在Manifest中的，这就导致了扩展性较差。除此之外，使用原生的路由方案会出现跳转过程无法控制的问题，因为一旦使用了startActivity()就无法插手其中任何环节了，只能交给系统管理，这就导致了在跳转失败的情况下无法降级，而是会直接抛出运营级的异常。这时候如果考虑使用自定义的路由组件就可以解决以上问题，比如通过URL索引就可以解决类依赖的问题；通过分布式管理页面配置可以解决隐式intent中集中式管理Path的问题；自己实现整个路由过程也可以拥有良好的扩展性，还可以通过AOP的方式解决跳转过程无法控制的问题，与此同时也能够提供非常灵活的降级方式。

#### 1、支持的功能

- 支持直接解析标准URL进行跳转，并自动注入参数到目标页面中
- 支持多模块工程使用
- 支持添加多个拦截器，自定义拦截顺序
- 支持依赖注入，可单独作为依赖注入框架使用
- 支持InstantRun
- 支持MultiDex(Google方案)
- 映射关系按组分类、多级管理，按需初始化
- 支持用户指定全局降级与局部降级策略
- 页面、拦截器、服务等组件均自动注册到框架
- 支持多种方式配置转场动画
- 支持获取Fragment
- 完全支持Kotlin以及混编(配置见文末 其他#5)
- 支持第三方 App 加固(使用 arouter-register 实现自动注册)
- 支持生成路由文档
- 提供 IDE 插件便捷的关联路径和目标类

#### 2、典型应用

- 从外部URL映射到内部页面，以及参数传递与解析
- 跨模块页面跳转，模块间解耦
- 拦截跳转过程，处理登陆、埋点等逻辑
- 跨模块API调用，通过控制反转来做组件解耦

以上介绍来自于 ARouter 的 Github 官网：[README_CN](https://github.com/alibaba/ARouter/blob/master/README_CN.md)

本文就基于其当前（2020/10/04）ARouter 的最新版本，对 ARouter 进行一次全面的源码解析和原理介绍，做到知其然也知其所以然，希望对你有所帮助😁😁

```groovy
dependencies {
    implementation 'com.alibaba:arouter-api:1.5.0'
    kapt 'com.alibaba:arouter-compiler:1.2.2'
}
```

### 二、前言

假设存在一个包含多个 moudle 的项目，在名为 **user** 的 moudle 中存在一个 `UserHomeActivity`，其对应的**路由路径**是 `/account/userHome`。那么，当我们要从其它 moudle 跳转到该页面时，只需要指定 path 来跳转即可

```java
package github.leavesc.user

@Route(path = RoutePath.USER_HOME)
class UserHomeActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_home)
    }

}

//其它页面使用如下代码来跳转到 UserHomeActivity
ARouter.getInstance().build(RoutePath.USER_HOME).navigation()
```

只根据一个 path，ARouter 是如何定位到特定的 Activity 的呢？

这就需要通过在**编译阶段**生成辅助代码来实现了。我们都知道，想要跳转到某个 Activity，那么就需要拿到该 Activity 的 Class 对象才行。在编译阶段，ARouter 会根据我们设定的路由跳转规则来自动生成映射文件，映射文件中就包含了 path 和 ActivityClass 之间的对应关系

例如，对于 `UserHomeActivity`，在编译阶段就会自动生成以下辅助文件。可以看到，`ARouter$$Group$$account` 类中就将 path 和 ActivityClass 作为键值对保存到了 Map 中。ARouter 就是依靠此来进行跳转的

```java
package com.alibaba.android.arouter.routes;

public class ARouter$$Group$$account implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/account/userHome", RouteMeta.build(RouteType.ACTIVITY, UserHomeActivity.class, "/account/userhome", "account", null, -1, -2147483648));
  }
}
```

还有一个重点需要注意，就是这类自动生成的文件的包名路径都是 `com.alibaba.android.arouter.routes`，且类名前缀也是有特定规则的。虽然 `ARouter$$Group$$account` 类实现了将对应关系保存到 Map 的逻辑，但是 `loadInto` 方法还是需要由 ARouter 在运行时来调用，那么 ARouter 就需要拿到 `ARouter$$Group$$account` 这个类才行，而 ARouter 就是通过扫描 `com.alibaba.android.arouter.routes`这个包名路径来获取所有辅助文件的

ARouter 的基本实现思路就是：

1. 开发者自己维护**特定 path** 和**特定的目标类**之间的对应关系，ARouter 只要求开发者使用包含了 path 的 `@Route` 注解修饰**目标类**
2. ARouter 在**编译阶段**通过**注解处理器**来自动生成 path 和特定的目标类之间的对应关系，即将 path 作为 key，将目标类的 Class 对象作为 value 之一存到 Map 之中
3. 在运行阶段，应用通过 path 来发起请求，ARouter 根据 path 从 Map 中取值，从而拿到目标类

### 三、初始化

ARouter 的一般是放在 `Application` 中调用 `init` 方法来完成初始化的，这里先来看下其初始化流程

```java
class MyApp : Application() {

    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            ARouter.openDebug()
            ARouter.openLog()
        }
        ARouter.init(this)
    }

}
```

`ARouter` 类使用了单例模式，逻辑比较简单，因为 `ARouter` 类只是负责对外暴露可以由外部调用的 API，大部分的实现逻辑还是转交由 `_ARouter` 类来完成

```java
public final class ARouter {

    private volatile static ARouter instance = null;

    private ARouter() {
    }

    /**
     * Get instance of router. A
     * All feature U use, will be starts here.
     */
    public static ARouter getInstance() {
        if (!hasInit) {
            throw new InitException("ARouter::Init::Invoke init(context) first!");
        } else {
            if (instance == null) {
                synchronized (ARouter.class) {
                    if (instance == null) {
                        instance = new ARouter();
                    }
                }
            }
            return instance;
        }
    }

    /**
     * Init, it must be call before used router.
     */
    public static void init(Application application) {
        if (!hasInit) { //防止重复初始化
            logger = _ARouter.logger;
            _ARouter.logger.info(Consts.TAG, "ARouter init start.");
            //通过 _ARouter 来完成初始化
            hasInit = _ARouter.init(application);
            if (hasInit) {
                _ARouter.afterInit();
            }
            _ARouter.logger.info(Consts.TAG, "ARouter init over.");
        }
    }

    ···

}
```

`_ARouter` 类是**包私有权限**，也使用了单例模式，其 `init(Application)` 方法的重点就在于 `LogisticsCenter.init(mContext, executor)`

```java
final class _ARouter {

    private volatile static _ARouter instance = null;

    private _ARouter() {
    }

    protected static _ARouter getInstance() {
        if (!hasInit) {
            throw new InitException("ARouterCore::Init::Invoke init(context) first!");
        } else {
            if (instance == null) {
                synchronized (_ARouter.class) {
                    if (instance == null) {
                        instance = new _ARouter();
                    }
                }
            }
            return instance;
        }
    }

    protected static synchronized boolean init(Application application) {
        mContext = application;
        //重点
        LogisticsCenter.init(mContext, executor);
        logger.info(Consts.TAG, "ARouter init success!");
        hasInit = true;
        mHandler = new Handler(Looper.getMainLooper());
        return true;
    }

    ···

}
```

`LogisticsCenter` 就实现了前文说的**扫描特定包名路径拿到所有自动生成的辅助文件**的逻辑，即在进行初始化的时候，我们就需要加载到当前项目一共包含的所有 group，以及每个 group 对应的路由信息表，其主要逻辑是：

1. 如果当前开启了 debug 模式或者通过本地 SP 缓存判断出 app 的版本前后发生了变化，那么就重新获取全局路由信息，否则就从使用之前缓存到 SP 中的数据
2. 获取全局路由信息是一个比较耗时的操作，所以 ARouter 就通过将全局路由信息缓存到 SP 中来实现复用。但由于在开发阶段开发者可能随时就会添加新的路由表，而每次发布新版本正常来说都是会加大应用的版本号的，所以 ARouter 就只在开启了 debug 模式或者是版本号发生了变化的时候才会重新获取路由信息
3. 获取到的路由信息中包含了在 `com.alibaba.android.arouter.routes` 这个包下自动生成的辅助文件的全路径，通过判断路径名的前缀字符串，就可以知道该类文件对应什么类型，然后通过反射构建不同类型的对象，通过调用对象的方法将路由信息存到 `Warehouse` 的 Map 中。至此，整个初始化流程就结束了

```java
public class LogisticsCenter {

    /**
     * LogisticsCenter init, load all metas in memory. Demand initialization
     */
    public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
        mContext = context;
        executor = tpe;

        try {
            long startInit = System.currentTimeMillis();
            //billy.qi modified at 2017-12-06
            //load by plugin first
            loadRouterMap();
            if (registerByPlugin) {
                logger.info(TAG, "Load router map by arouter-auto-register plugin.");
            } else {
                Set<String> routerMap;

                //如果当前开启了 debug 模式或者通过本地 SP 缓存判断出 app 的版本前后发生了变化
                //那么就重新获取路由信息，否则就从使用之前缓存到 SP 中的数据
                // It will rebuild router map every times when debuggable.
                if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                    logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
                    // These class was generated by arouter-compiler.
                    //获取 ROUTE_ROOT_PAKCAGE 包名路径下包含的所有的 ClassName
                    routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                    if (!routerMap.isEmpty()) {
                        //缓存到 SP 中
                        context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                    }
                    //更新 App 的版本信息
                    PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
                } else {
                    logger.info(TAG, "Load router map from cache.");
                    routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
                }

                logger.info(TAG, "Find router map finished, map size = " + routerMap.size() + ", cost " + (System.currentTimeMillis() - startInit) + " ms.");
                startInit = System.currentTimeMillis();

                for (String className : routerMap) {
                    //通过 className 的前缀来判断该 class 对应的什么类型，并同时缓存到 Warehouse 中
                    //1.IRouteRoot
                    //2.IInterceptorGroup
                    //3.IProviderGroup
                    if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                        // This one of root elements, load root.
                        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                        // Load interceptorMeta
                        ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                        // Load providerIndex
                        ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                    }
                }
            }

           ···

        } catch (Exception e) {
            throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
        }
    }

}
```

对于第三步，可以举个例子来加强理解。对于上文所讲的 `UserHomeActivity`，其对应的 path 是 `/account/userHome`，ARouter 默认会将 path 的第一个单词即 `account` 作为其 `group`，而且 `UserHomeActivity` 是放在名为 `user` 的 module 中

而 ARouter 在通过注解处理器生成辅助文件的时候，类名就会根据以上信息来生成，所以最终就会生成以下两个文件：

```java
package com.alibaba.android.arouter.routes;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Root$$user implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("account", ARouter$$Group$$account.class);
  }
}
```

```java
package com.alibaba.android.arouter.routes;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Group$$account implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/account/userHome", RouteMeta.build(RouteType.ACTIVITY, UserHomeActivity.class, "/account/userhome", "account", null, -1, -2147483648));
  }
}

```

`LogisticsCenter` 的 `init` 方法就会根据文件名的固定前缀 `ARouter$$Root$$` 定位到 `ARouter$$Root$$user` 这个类，然后通过反射构建出该对象，然后通过调用其 `loadInto` 方法将键值对保存到 `Warehouse.groupsIndex` 中。等到后续需要跳转到 `group` 为 `account` 的页面时，就会再来反射调用 `ARouter$$Group$$account` 的 `loadInto` 方法，即按需加载，等到需要的时候再来获取详细的路由对应信息

因为对于一个大型的 App 来说，可能包含一百或者几百个页面，如果一次性将所有路由信息都加载到内存中，对于内存的压力是比较大的，而用户每次使用可能也只会打开十几个页面，所以这里必须是按需加载

### 四、跳转到 Activity

讲完初始化流程，那就再来看下 ARouter 实现 Activity 跳转的流程

跳转到 Activity 最简单的方式就是只指定 path：

```kotlin
	ARouter.getInstance().build(RoutePath.USER_HOME).navigation()
```

`build()` 方法会通过 `ARouter` 中转调用到 `_ARouter` 的 `build()` 方法，最终返回一个 `Postcard` 对象

```java
    /**
     * Build postcard by path and default group
     */
    protected Postcard build(String path) {
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                //用于路径替换，这对于某些需要控制页面跳转流程的场景比较有用
                //例如，如果某个页面需要登录才可以展示的话
                //就可以通过 PathReplaceService 将 path 替换 loginPagePath
                path = pService.forString(path);
            }
            //使用字符串 path 包含的第一个单词作为 group
            return build(path, extractGroup(path));
        }
    }

    /**
     * Build postcard by path and group
     */
    protected Postcard build(String path, String group) {
        if (TextUtils.isEmpty(path) || TextUtils.isEmpty(group)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
            return new Postcard(path, group);
        }
    }
```

返回的 `Postcard` 对象可以用于传入一些跳转配置参数，例如：携带参数 `mBundle`、开启绿色通道 `greenChannel` 、跳转动画 `optionsCompat` 等

```java
public final class Postcard extends RouteMeta {
    // Base
    private Uri uri;
    private Object tag;             // A tag prepare for some thing wrong.
    private Bundle mBundle;         // Data to transform
    private int flags = -1;         // Flags of route
    private int timeout = 300;      // Navigation timeout, TimeUnit.Second
    private IProvider provider;     // It will be set value, if this postcard was provider.
    private boolean greenChannel;
    private SerializationService serializationService;

}
```

`Postcard` 的 `navigation()` 方法又会调用到 `_ARouter` 的以下方法来完成 Activity 的跳转。该方法逻辑上并不复杂，注释也写得很清楚了

```java
final class _ARouter {

    /**
     * Use router navigation.
     *
     * @param context     Activity or null.
     * @param postcard    Route metas
     * @param requestCode RequestCode
     * @param callback    cb
     */
    protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
        if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
            // Pretreatment failed, navigation canceled.
            //用于执行跳转前的预处理操作，可以通过 onPretreatment 方法的返回值决定是否取消跳转
            return null;
        }

        try {
            LogisticsCenter.completion(postcard);
        } catch (NoRouteFoundException ex) {
            //没有找到匹配的目标类
            //下面就执行一些提示操作和事件回调通知

            logger.warning(Consts.TAG, ex.getMessage());
            if (debuggable()) {
                // Show friendly tips for user.
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(mContext, "There's no route matched!\n" +
                                " Path = [" + postcard.getPath() + "]\n" +
                                " Group = [" + postcard.getGroup() + "]", Toast.LENGTH_LONG).show();
                    }
                });
            }
            if (null != callback) {
                callback.onLost(postcard);
            } else {
                // No callback for this invoke, then we use the global degrade service.
                DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
                if (null != degradeService) {
                    degradeService.onLost(context, postcard);
                }
            }
            return null;
        }

        if (null != callback) {
            //找到了匹配的目标类
            callback.onFound(postcard);
        }

        if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
            //没有开启绿色通道，那么就还需要执行所有拦截器
            //外部可以通过拦截器实现：控制是否允许跳转、更改跳转参数等逻辑

            interceptorService.doInterceptions(postcard, new InterceptorCallback() {
                /**
                 * Continue process
                 *
                 * @param postcard route meta
                 */
                @Override
                public void onContinue(Postcard postcard) {
                    //拦截器允许跳转
                    _navigation(context, postcard, requestCode, callback);
                }

                /**
                 * Interrupt process, pipeline will be destory when this method called.
                 *
                 * @param exception Reson of interrupt.
                 */
                @Override
                public void onInterrupt(Throwable exception) {
                    if (null != callback) {
                        callback.onInterrupt(postcard);
                    }

                    logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
                }
            });
        } else {
            //开启了绿色通道，直接跳转，不需要遍历拦截器
            return _navigation(context, postcard, requestCode, callback);
        }

        return null;
    }

    //由于本例子的目标页面是 Activity，所以只看 ACTIVITY 即可
    private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;
        switch (postcard.getType()) {
            case ACTIVITY:
                // Build intent
                //Destination 就是指向目标 Activity 的 class 对象
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                //塞入携带的参数
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Set Actions
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }

                // Navigation in main looper.
                //最终在主线程完成跳转
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });

                break;
            ··· //省略其它类型判断
        }

        return null;
    }

}
```

`navigation` 方法的重点在于 `LogisticsCenter.completion(postcard)` 这一句代码。在讲 ARouter 初始化流程的时候有讲到：等到后续需要跳转到 `group` 为 `account` 的页面时，就会再来反射调用 `ARouter$$Group$$account` 的 `loadInto` 方法，即按需加载，等到需要的时候再来获取详细的路由对应信息

`completion` 方法就是用来获取详细的路由对应信息的。该方法会通过 `postcard` 携带的 path 和 group 信息从 `Warehouse` 取值，如果值不为 null 的话就将信息保存到 `postcard` 中，如果值为 null 的话就抛出 `NoRouteFoundException`

```java
	/**
     * Completion the postcard by route metas
     *
     * @param postcard Incomplete postcard, should complete by this method.
     */
    public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }

        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {    //为 null 说明目标类不存在或者是该 group 还未加载过
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
            if (null == groupMeta) {
                //groupMeta 为 null，说明 postcard 的 path 对应的 group 不存在，抛出异常
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                // Load route and cache it into memory, then delete from metas.
                try {
                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] starts loading, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }
				  //会执行到这里，说明此 group 还未加载过，那么就来反射加载 group 对应的所有 path 信息
                   //获取后就保存到 Warehouse.routes
                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    iGroupInstance.loadInto(Warehouse.routes);

                    //移除此 group
                    Warehouse.groupsIndex.remove(postcard.getGroup());

                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }
                } catch (Exception e) {
                    throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
                }

                //重新执行一遍
                completion(postcard);   // Reload
            }
        } else {
            //拿到详细的路由信息了，将这些信息存到 postcard 中

            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());

            //省略一些和本例子无关的代码
            ···
        }
    }
```

### 五、跳转到 Activity 并注入参数

ARouter 也支持在跳转到 Activity 的同时向目标页面自动注入参数

在跳转的时候指定要携带的键值对参数：

```kotlin
ARouter.getInstance().build(RoutePath.USER_HOME)
        .withLong(RoutePath.USER_HOME_PARAMETER_ID, 20)
        .withString(RoutePath.USER_HOME_PARAMETER_NAME, "leavesC")
        .navigation()

object RoutePath {

    const val USER_HOME = "/account/userHome"

    const val USER_HOME_PARAMETER_ID = "userHomeId"

    const val USER_HOME_PARAMETER_NAME = "userName"

}
```

在目标页面通过 `@Autowired` 注解修饰变量。注解可以同时声明其 `name` 参数，用于和传递的键值对中的 key 对应上，这样 ARouter 才知道应该向哪个变量赋值。如果没有声明 `name` 参数，那么 `name` 参数就默认和**变量名**相等

这样，在我们调用 `ARouter.getInstance().inject(this)` 后，ARouter 就会自动完成参数的赋值

```kotlin
package github.leavesc.user

/**
 * 作者：leavesC
 * 时间：2020/10/4 14:05
 * 描述：
 * GitHub：https://github.com/leavesC
 */
@Route(path = RoutePath.USER_HOME)
class UserHomeActivity : AppCompatActivity() {

    @Autowired(name = RoutePath.USER_HOME_PARAMETER_ID)
    @JvmField
    var userId: Long = 0

    @Autowired
    @JvmField
    var userName = ""

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_home)
        ARouter.getInstance().inject(this)
        tv_hint.text = "$userId $userName"
    }

}
```

ARouter 实现参数自动注入也需要依靠注解处理器生成的辅助文件来实现，即会生成以下的辅助代码：

```java
package github.leavesc.user;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class UserHomeActivity$$ARouter$$Autowired implements ISyringe {

  //用于实现序列化和反序列化
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    serializationService = ARouter.getInstance().navigation(SerializationService.class);
    UserHomeActivity substitute = (UserHomeActivity)target;
    substitute.userId = substitute.getIntent().getLongExtra("userHomeId", substitute.userId);
    substitute.userName = substitute.getIntent().getStringExtra("userName");
  }
}

```

因为在跳转到 Activity 时携带的参数也是需要放到 `Intent` 里的，所以 `inject` 方法也只是帮我们实现了从 `Intent` 取值然后向变量赋值的逻辑而已，这就要求相应的变量必须是 `public` 的，这就是在 Kotlin 代码中需要同时向变量加上 `@JvmField`注解的原因

现在来看下 ARouter 是如何实现**参数自动注入**的，其起始方法就是：`ARouter.getInstance().inject(this)`，其最终会调用到以下方法

```java
final class _ARouter {

    static void inject(Object thiz) {
        AutowiredService autowiredService = ((AutowiredService) ARouter.getInstance().build("/arouter/service/autowired").navigation());
        if (null != autowiredService) {
            autowiredService.autowire(thiz);
        }
    }

}
```

ARouter 通过控制反转的方式拿到 `AutowiredService` 对应的实现类 `AutowiredServiceImpl`的实例对象，然后调用其 `autowire` 方法完成参数注入

由于生成的**参数注入辅助类**的类名具有**固定的包名和类名**，即包名和目标类所在包名一致，类名是**目标类类名**+ `$$ARouter$$Autowired`，所以在 `AutowiredServiceImpl` 中就可以根据传入的 `instance` 参数和反射来生成辅助类对象，最终调用其 `inject` 方法完成参数注入

```java
@Route(path = "/arouter/service/autowired")
public class AutowiredServiceImpl implements AutowiredService {
    private LruCache<String, ISyringe> classCache;
    private List<String> blackList;

    @Override
    public void init(Context context) {
        classCache = new LruCache<>(66);
        blackList = new ArrayList<>();
    }

    @Override
    public void autowire(Object instance) {
        String className = instance.getClass().getName();
        try {
            //如果在白名单中了的话，那么就不再执行参数注入
            if (!blackList.contains(className)) {
                ISyringe autowiredHelper = classCache.get(className);
                if (null == autowiredHelper) {  // No cache.
                    autowiredHelper = (ISyringe) Class.forName(instance.getClass().getName() + SUFFIX_AUTOWIRED).getConstructor().newInstance();
                }
                //完成参数注入
                autowiredHelper.inject(instance);
                //缓存起来，避免重复反射
                classCache.put(className, autowiredHelper);
            }
        } catch (Exception ex) {
            //如果参数注入过程抛出异常，那么就将其加入白名单中
            blackList.add(className);    // This instance need not autowired.
        }
    }
}
```

### 六、控制反转

上一节所讲的**跳转到 Activity 并自动注入参数**属于**依赖注入**的一种，ARouter 同时也支持**控制反转**：通过接口来获取其实现类实例

例如，假设存在一个 `ISayHelloService` 接口，我们需要拿到其实现类实例，但是不希望在使用的时候和特定的实现类 `SayHelloService` 绑定在一起从而造成强耦合，此时就可以使用 ARouter 的控制反转功能，但这也要求 `ISayHelloService` 接口继承了 `IProvider` 接口才行

```kotlin
/**
 * 作者：leavesC
 * 时间：2020/10/4 13:49
 * 描述：
 * GitHub：https://github.com/leavesC
 */
interface ISayHelloService : IProvider {

    fun sayHello()

}

@Route(path = RoutePath.SERVICE_SAY_HELLO)
class SayHelloService : ISayHelloService {

    override fun init(context: Context) {

    }

    override fun sayHello() {
        Log.e("SayHelloService", "$this sayHello")
    }

}
```

在使用的时候直接传递 `ISayHelloService` 的 Class 对象即可，ARouter 会将 `SayHelloService` 以单例模式的形式返回，无需开发者手动去构建 `SayHelloService` 对象，从而达到解耦的目的

```kotlin
ARouter.getInstance().navigation(ISayHelloService::class.java).sayHello()
```

和实现 Activity 跳转的时候一样，ARouter 也会自动生成以下几个文件，包含了路由表的映射关系

```java
package com.alibaba.android.arouter.routes;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Group$$account implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/account/sayHelloService", RouteMeta.build(RouteType.PROVIDER, SayHelloService.class, "/account/sayhelloservice", "account", null, -1, -2147483648));
  }
}

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Providers$$user implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
    providers.put("github.leavesc.user.ISayHelloService", RouteMeta.build(RouteType.PROVIDER, SayHelloService.class, "/account/sayHelloService", "account", null, -1, -2147483648));
  }
}

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Root$$user implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("account", ARouter$$Group$$account.class);
  }
}
```

这里再来看下其具体的实现原理

在讲初始化流程的时候有讲到，`LogisticsCenter` 实现了**扫描特定包名路径拿到所有自动生成的辅助文件**的逻辑。所以，最终 Warehouse 中就会在初始化的时候拿到以下数据

Warehouse.groupsIndex：

- `account` -> `class com.alibaba.android.arouter.routes.ARouter$$Group$$account`

Warehouse.providersIndex：

- `github.leavesc.user.ISayHelloService` -> `RouteMeta.build(RouteType.PROVIDER, SayHelloService.class, "/account/sayHelloService", "account", null, -1, -2147483648)`

`ARouter.getInstance().navigation(ISayHelloService::class.java)` 最终会中转调用到 `_ARouter` 的以下方法

```java
	protected <T> T navigation(Class<? extends T> service) {
        try {
            //从 Warehouse.providersIndex 取值拿到 RouteMeta 中存储的 path 和 group
            Postcard postcard = LogisticsCenter.buildProvider(service.getName());

            // Compatible 1.0.5 compiler sdk.
            // Earlier versions did not use the fully qualified name to get the service
            if (null == postcard) {
                // No service, or this service in old version.
                postcard = LogisticsCenter.buildProvider(service.getSimpleName());
            }

            if (null == postcard) {
                return null;
            }
		   //重点
            LogisticsCenter.completion(postcard);
            return (T) postcard.getProvider();
        } catch (NoRouteFoundException ex) {
            logger.warning(Consts.TAG, ex.getMessage());
            return null;
        }
    }
```

`LogisticsCenter.completion(postcard)` 方法的流程和之前讲解的差不多，只是在获取对象实例的时候同时将实例缓存起来，留待之后复用，至此就完成了控制反转的流程了

```java
	/**
     * Completion the postcard by route metas
     *
     * @param postcard Incomplete postcard, should complete by this method.
     */
    public synchronized static void completion(Postcard postcard) {
	   ... //省略之前已经讲解过的代码

	   RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());

        switch (routeMeta.getType()) {
                case PROVIDER:  // if the route is provider, should find its instance
                    // Its provider, so it must implement IProvider
                	//拿到 SayHelloService Class 对象
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { // There's no instance of this provider
                        //instance 等于 null 说明是第一次取值
                        //那么就通过反射构建 SayHelloService 对象，然后将之缓存到 Warehouse.providers 中
                        //所以通过控制反转获取的对象在应用的整个生命周期内只会有一个实例
                        IProvider provider;
                        try {
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            throw new HandlerException("Init provider failed! " + e.getMessage());
                        }
                    }
                	//将获取到的实例存起来
                    postcard.setProvider(instance);
                    postcard.greenChannel();    // Provider should skip all of interceptors
                    break;
                case FRAGMENT:
                    postcard.greenChannel();    // Fragment needn't interceptors
                default:
                    break;
            }

    }
```

### 七、拦截器

ARouter 的拦截器对于某些需要控制页面跳转流程的业务逻辑来说是十分有用的功能。例如，用户如果要跳转到个人资料页面时，我们就可以通过拦截器来判断用户是否处于已登录状态，还未登录的话就可以拦截该请求，然后自动为用户打开登录页面

我们可以同时设定多个拦截器，每个拦截器设定不同的优先级

```kotlin
/**
 * 作者：leavesC
 * 时间：2020/10/5 11:49
 * 描述：
 * GitHub：https://github.com/leavesC
 */
@Interceptor(priority = 100, name = "啥也不做的拦截器")
class NothingInterceptor : IInterceptor {

    override fun init(context: Context) {

    }

    override fun process(postcard: Postcard, callback: InterceptorCallback) {
        //不拦截，任其跳转
        callback.onContinue(postcard)
    }

}

@Interceptor(priority = 200, name = "登陆拦截器")
class LoginInterceptor : IInterceptor {

    override fun init(context: Context) {

    }

    override fun process(postcard: Postcard, callback: InterceptorCallback) {
        if (postcard.path == RoutePath.USER_HOME) {
            //拦截
            callback.onInterrupt(null)
            //跳转到登陆页
            ARouter.getInstance().build(RoutePath.USER_LOGIN).navigation()
        } else {
            //不拦截，任其跳转
            callback.onContinue(postcard)
        }
    }

}
```

这样，当我们执行 `ARouter.getInstance().build(RoutePath.USER_HOME).navigation()` 想要跳转的时候，就会发现打开的其实是登录页 `RoutePath.USER_LOGIN`

来看下拦截器是如何实现的

对于以上的两个拦截器，会生成以下的辅助文件。辅助文件会拿到所有我们自定义的拦截器实现类并根据优先级高低存到 Map 中

```java
package com.alibaba.android.arouter.routes;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Interceptors$$user implements IInterceptorGroup {
  @Override
  public void loadInto(Map<Integer, Class<? extends IInterceptor>> interceptors) {
    interceptors.put(100, NothingInterceptor.class);
    interceptors.put(200, LoginInterceptor.class);
  }
}
```

而这些拦截器一样是会在初始化的时候，通过`LogisticsCenter.init`方法存到 `Warehouse.interceptorsIndex`中

```java
 /**
     * LogisticsCenter init, load all metas in memory. Demand initialization
     */
    public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {

        ···

        for (String className : routerMap) {
                    if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                        // This one of root elements, load root.
                        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                        // Load interceptorMeta
                        //拿到自定义的拦截器实现类
                        ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                        // Load providerIndex
                        ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                    }
                }

          ···

    }
```

然后，在 `_ARouter` 的 `navigation` 方法中，如何判断到此次路由请求没有开启绿色通道模式的话，那么就会将此次请求转交给 `interceptorService`，让其去遍历每个拦截器

```java
final class _ARouter {

    /**
     * Use router navigation.
     *
     * @param context     Activity or null.
     * @param postcard    Route metas
     * @param requestCode RequestCode
     * @param callback    cb
     */
    protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {

        ···

        if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.

            //遍历拦截器
            interceptorService.doInterceptions(postcard, new InterceptorCallback() {
                /**
                 * Continue process
                 *
                 * @param postcard route meta
                 */
                @Override
                public void onContinue(Postcard postcard) {
                    _navigation(context, postcard, requestCode, callback);
                }

                /**
                 * Interrupt process, pipeline will be destory when this method called.
                 *
                 * @param exception Reson of interrupt.
                 */
                @Override
                public void onInterrupt(Throwable exception) {
                    if (null != callback) {
                        callback.onInterrupt(postcard);
                    }

                    logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
                }
            });
        } else {
            return _navigation(context, postcard, requestCode, callback);
        }

        return null;
    }

}
```

`interceptorService` 变量属于 `InterceptorService` 接口类型，该接口的实现类是 `InterceptorServiceImpl`，ARouter内部在初始化的过程中也是根据控制反转的方式来拿到 `interceptorService` 这个实例的

`InterceptorServiceImpl` 的主要逻辑是：

1. 在第一次获取 `InterceptorServiceImpl` 实例的时候，其 `init` 方法会马上被调用，该方法内部会交由线程池来执行，通过反射生成每个拦截器对象，并调用每个拦截器的 `init` 方法来完成拦截器的初始化，并将每个拦截器对象都存到 `Warehouse.interceptors` 中。如果初始化完成了，则唤醒等待在 `interceptorInitLock` 上的线程
2. 当拦截器逻辑被触发，即 `doInterceptions` 方法被调用时，如果此时第一个步骤还未执行完的话，则会通过 `checkInterceptorsInitStatus()`方法等待第一个步骤执行完成。如果十秒内都未完成的话，则走失败流程直接返回
3. 在线程池中遍历拦截器列表，如果有某个拦截器拦截了请求的话则调用 `callback.onInterrupt`方法通知外部，否则的话则调用 `callback.onContinue()` 方法继续跳转逻辑

```java
@Route(path = "/arouter/service/interceptor")
public class InterceptorServiceImpl implements InterceptorService {
    private static boolean interceptorHasInit;
    private static final Object interceptorInitLock = new Object();

    @Override
    public void init(final Context context) {
        LogisticsCenter.executor.execute(new Runnable() {
            @Override
            public void run() {
                if (MapUtils.isNotEmpty(Warehouse.interceptorsIndex)) {
                    //遍历拦截器列表，通过反射构建对象并初始化
                    for (Map.Entry<Integer, Class<? extends IInterceptor>> entry : Warehouse.interceptorsIndex.entrySet()) {
                        Class<? extends IInterceptor> interceptorClass = entry.getValue();
                        try {
                            IInterceptor iInterceptor = interceptorClass.getConstructor().newInstance();
                            iInterceptor.init(context);
                            //存起来
                            Warehouse.interceptors.add(iInterceptor);
                        } catch (Exception ex) {
                            throw new HandlerException(TAG + "ARouter init interceptor error! name = [" + interceptorClass.getName() + "], reason = [" + ex.getMessage() + "]");
                        }
                    }

                    interceptorHasInit = true;

                    logger.info(TAG, "ARouter interceptors init over.");

                    synchronized (interceptorInitLock) {
                        interceptorInitLock.notifyAll();
                    }
                }
            }
        });
    }


    @Override
    public void doInterceptions(final Postcard postcard, final InterceptorCallback callback) {
        if (null != Warehouse.interceptors && Warehouse.interceptors.size() > 0) {

            checkInterceptorsInitStatus();

            if (!interceptorHasInit) {
                //初始化太久，不等了，直接走失败流程
                callback.onInterrupt(new HandlerException("Interceptors initialization takes too much time."));
                return;
            }

            LogisticsCenter.executor.execute(new Runnable() {
                @Override
                public void run() {
                    CancelableCountDownLatch interceptorCounter = new CancelableCountDownLatch(Warehouse.interceptors.size());
                    try {
                        _excute(0, interceptorCounter, postcard);
                        interceptorCounter.await(postcard.getTimeout(), TimeUnit.SECONDS);
                        if (interceptorCounter.getCount() > 0) {    // Cancel the navigation this time, if it hasn't return anythings.
                            //大于 0 说明此次请求被某个拦截器拦截了，走失败流程
                            callback.onInterrupt(new HandlerException("The interceptor processing timed out."));
                        } else if (null != postcard.getTag()) {    // Maybe some exception in the tag.
                            callback.onInterrupt(new HandlerException(postcard.getTag().toString()));
                        } else {
                            callback.onContinue(postcard);
                        }
                    } catch (Exception e) {
                        callback.onInterrupt(e);
                    }
                }
            });
        } else {
            callback.onContinue(postcard);
        }
    }

    /**
     * Excute interceptor
     *
     * @param index    current interceptor index
     * @param counter  interceptor counter
     * @param postcard routeMeta
     */
    private static void _excute(final int index, final CancelableCountDownLatch counter, final Postcard postcard) {
        if (index < Warehouse.interceptors.size()) {
            IInterceptor iInterceptor = Warehouse.interceptors.get(index);
            iInterceptor.process(postcard, new InterceptorCallback() {
                @Override
                public void onContinue(Postcard postcard) {
                    // Last interceptor excute over with no exception.
                    counter.countDown();
                    _excute(index + 1, counter, postcard);  // When counter is down, it will be execute continue ,but index bigger than interceptors size, then U know.
                }

                @Override
                public void onInterrupt(Throwable exception) {
                    // Last interceptor excute over with fatal exception.

                    postcard.setTag(null == exception ? new HandlerException("No message.") : exception.getMessage());    // save the exception message for backup.
                    counter.cancel();
                    // Be attention, maybe the thread in callback has been changed,
                    // then the catch block(L207) will be invalid.
                    // The worst is the thread changed to main thread, then the app will be crash, if you throw this exception!
//                    if (!Looper.getMainLooper().equals(Looper.myLooper())) {    // You shouldn't throw the exception if the thread is main thread.
//                        throw new HandlerException(exception.getMessage());
//                    }
                }
            });
        }
    }

    private static void checkInterceptorsInitStatus() {
        synchronized (interceptorInitLock) {
            while (!interceptorHasInit) {
                try {
                    interceptorInitLock.wait(10 * 1000);
                } catch (InterruptedException e) {
                    throw new HandlerException(TAG + "Interceptor init cost too much time error! reason = [" + e.getMessage() + "]");
                }
            }
        }
    }

}
```

### 八、注解处理器

通篇读下来，读者应该能够感受到注解处理器在 ARouter 中起到了很大的作用，依靠注解处理器生成的辅助文件，ARouter 才能完成**参数自动注入**等功能。这里就再来介绍下 ARouter 关于注解处理器的实现原理

APT(**Annotation Processing Tool**) 即注解处理器，是一种注解处理工具，用来在编译期扫描和处理注解，通过注解来生成 Java 文件。即以注解作为桥梁，通过预先规定好的代码生成规则来自动生成 Java 文件。此类注解框架的代表有 **ButterKnife、Dragger2、EventBus** 等

Java API 已经提供了扫描源码并解析注解的框架，开发者可以通过继承 **AbstractProcessor** 类来实现自己的注解解析逻辑。APT 的原理就是在注解了某些代码元素（如字段、函数、类等）后，在编译时编译器会检查 **AbstractProcessor** 的子类，并且自动调用其 **process()** 方法，然后将添加了指定注解的所有代码元素作为参数传递给该方法，开发者再根据注解元素在编译期输出对应的 Java 代码

关于 APT 技术的原理和应用可以看这篇文章：[Android APT 实例讲解](https://juejin.im/post/6844903753108160525)

ARouter 源码中和注解处理器相关的 module 有两个：

- arouter-annotation。Java Module，包含了像 Autowired、Interceptor 这些注解以及 RouteMeta 等 JavaBean
- arouter-compiler。Android Module，包含了多个 AbstractProcessor 的实现类用于生成代码

这里主要来看 `arouter-compiler`，这里以自定义的拦截器 `NothingInterceptor` 作为例子

```kotlin
package github.leavesc.user

/**
 * 作者：leavesC
 * 时间：2020/10/5 11:49
 * 描述：
 * GitHub：https://github.com/leavesC
 */
@Interceptor(priority = 100, name = "啥也不做的拦截器")
class NothingInterceptor : IInterceptor {

    override fun init(context: Context) {

    }

    override fun process(postcard: Postcard, callback: InterceptorCallback) {
        //不拦截，任其跳转
        callback.onContinue(postcard)
    }

}
```

生成的辅助文件：

```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.template.IInterceptor;
import com.alibaba.android.arouter.facade.template.IInterceptorGroup;
import github.leavesc.user.NothingInterceptor;
import java.lang.Class;
import java.lang.Integer;
import java.lang.Override;
import java.util.Map;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Interceptors$$user implements IInterceptorGroup {
  @Override
  public void loadInto(Map<Integer, Class<? extends IInterceptor>> interceptors) {
    interceptors.put(100, NothingInterceptor.class);
  }
}
```

那么，生成的辅助文件我们就要包含以下几个元素：

1. 包名
2. 导包
3. 注释
4. 实现类及继承的接口
5. 包含的方法及方法参数
6. 方法体
7. 修饰符

如果通过硬编码的形式，即通过拼接字符串的方式来生成以上代码也是可以的，但是这样会使得代码不好维护且可读性很低，所以 ARouter 是通过 `JavaPoet` 这个开源库来生成代码的。`JavaPoet` 是 square 公司开源的 Java 代码生成框架，可以很方便地通过其提供的 API 来生成指定格式（修饰符、返回值、参数、函数体等）的代码

拦截器对应的 `AbstractProcessor` 子类就是 `InterceptorProcessor`，其主要逻辑是：

1. 在 process 方法中通过 RoundEnvironment 拿到所有使用了 @Interceptor 注解进行修饰的代码元素 elements，然后遍历所有 item
2. 判断每个 item 是否继承了 IInterceptor 接口，是的话则说明该 item 就是我们要找的拦截器实现类
3. 获取每个 item 包含的 @Interceptor 注解对象，根据我们为之设定的优先级 priority，将每个 item 按顺序存到 interceptors 中
4. 如果存在两个拦截器的优先级相同，那么就抛出异常
5. 将所有拦截器按顺序存入 interceptors 后，通过 JavaPoet 提供的 API 来生成**包名、导包、注释、实现类**等多个代码元素，并最终生成一个完整的类文件

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes(ANNOTATION_TYPE_INTECEPTOR)
public class InterceptorProcessor extends BaseProcessor {

    //用于保存拦截器，按照优先级高低进行排序
    private Map<Integer, Element> interceptors = new TreeMap<>();

    private TypeMirror iInterceptor = null;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);

        iInterceptor = elementUtils.getTypeElement(Consts.IINTERCEPTOR).asType();

        logger.info(">>> InterceptorProcessor init. <<<");
    }

    /**
     * {@inheritDoc}
     *
     * @param annotations
     * @param roundEnv
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (CollectionUtils.isNotEmpty(annotations)) {
            //拿到所有使用了 @Interceptor 进行修饰的代码元素
            Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(Interceptor.class);
            try {
                parseInterceptors(elements);
            } catch (Exception e) {
                logger.error(e);
            }
            return true;
        }

        return false;
    }

    /**
     * Parse tollgate.
     *
     * @param elements elements of tollgate.
     */
    private void parseInterceptors(Set<? extends Element> elements) throws IOException {
        if (CollectionUtils.isNotEmpty(elements)) {
            logger.info(">>> Found interceptors, size is " + elements.size() + " <<<");

            // Verify and cache, sort incidentally.
            for (Element element : elements) {
                //判断使用了 @Interceptor 进行修饰的代码元素是否同时实现了 com.alibaba.android.arouter.facade.template.IInterceptor 这个接口
                //两者缺一不可
                if (verify(element)) {  // Check the interceptor meta
                    logger.info("A interceptor verify over, its " + element.asType());
                    Interceptor interceptor = element.getAnnotation(Interceptor.class);

                    Element lastInterceptor = interceptors.get(interceptor.priority());
                    if (null != lastInterceptor) { // Added, throw exceptions
                        //不为 null 说明存在两个拦截器其优先级相等，这是不允许的，直接抛出异常
                        throw new IllegalArgumentException(
                                String.format(Locale.getDefault(), "More than one interceptors use same priority [%d], They are [%s] and [%s].",
                                        interceptor.priority(),
                                        lastInterceptor.getSimpleName(),
                                        element.getSimpleName())
                        );
                    }
                    //将拦截器按照优先级高低进行排序保存
                    interceptors.put(interceptor.priority(), element);
                } else {
                    logger.error("A interceptor verify failed, its " + element.asType());
                }
            }

            // Interface of ARouter.
            //拿到 com.alibaba.android.arouter.facade.template.IInterceptor 这个接口的类型抽象
            TypeElement type_ITollgate = elementUtils.getTypeElement(IINTERCEPTOR);
            //拿到 com.alibaba.android.arouter.facade.template.IInterceptorGroup 这个接口的类型抽象
            TypeElement type_ITollgateGroup = elementUtils.getTypeElement(IINTERCEPTOR_GROUP);

            /**
             *  Build input type, format as :
             *
             *  ```Map<Integer, Class<? extends ITollgate>>```
             */
            //生成对 Map<Integer, Class<? extends IInterceptor>> 这段代码的抽象封装
            ParameterizedTypeName inputMapTypeOfTollgate = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(Integer.class),
                    ParameterizedTypeName.get(
                            ClassName.get(Class.class),
                            WildcardTypeName.subtypeOf(ClassName.get(type_ITollgate))
                    )
            );

            // Build input param name.
            //生成 loadInto 方法的入参参数 interceptors
            ParameterSpec tollgateParamSpec = ParameterSpec.builder(inputMapTypeOfTollgate, "interceptors").build();

            // Build method : 'loadInto'
            //生成 loadInto 方法
            MethodSpec.Builder loadIntoMethodOfTollgateBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(tollgateParamSpec);

            // Generate
            if (null != interceptors && interceptors.size() > 0) {
                // Build method body
                for (Map.Entry<Integer, Element> entry : interceptors.entrySet()) {
                    //遍历每个拦截器，生成 interceptors.put(100, NothingInterceptor.class); 这类型的代码
                    loadIntoMethodOfTollgateBuilder.addStatement("interceptors.put(" + entry.getKey() + ", $T.class)", ClassName.get((TypeElement) entry.getValue()));
                }
            }

            // Write to disk(Write file even interceptors is empty.)
            //包名固定是 PACKAGE_OF_GENERATE_FILE，即 com.alibaba.android.arouter.routes
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                    TypeSpec.classBuilder(NAME_OF_INTERCEPTOR + SEPARATOR + moduleName) //设置类名
                            .addModifiers(PUBLIC) //添加 public 修饰符
                            .addJavadoc(WARNING_TIPS) //添加注释
                            .addMethod(loadIntoMethodOfTollgateBuilder.build()) //添加 loadInto 方法
                            .addSuperinterface(ClassName.get(type_ITollgateGroup)) //最后生成的类同时实现了 IInterceptorGroup 接口
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Interceptor group write over. <<<");
        }
    }

    /**
     * Verify inteceptor meta
     *
     * @param element Interceptor taw type
     * @return verify result
     */
    private boolean verify(Element element) {
        Interceptor interceptor = element.getAnnotation(Interceptor.class);
        // It must be implement the interface IInterceptor and marked with annotation Interceptor.
        return null != interceptor && ((TypeElement) element).getInterfaces().contains(iInterceptor);
    }
}
```

### 九、结尾

ARouter 的实现原理和源码解析都讲得差不多了，文本应该讲得挺全面的了，那么下一篇就再来进入实战篇吧，自己来动手实现一个 ARouter 😁😁
