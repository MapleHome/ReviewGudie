## MMKV

官网: https://github.com/Tencent/MMKV
MMKV 原理: https://github.com/Tencent/MMKV/wiki/design
Android 进阶教程: https://github.com/Tencent/MMKV/wiki/android_advance_cn
Android 多进程设计与实现: https://github.com/Tencent/MMKV/wiki/android_ipc
Android 性能对比: https://github.com/Tencent/MMKV/wiki/android_benchmark_cn

MMKV 是基于 mmap 内存映射的 key-value 组件，底层序列化/反序列化使用 protobuf 实现，性能高，稳定性强。从 2015 年中至今在微信上使用，其性能和稳定性经过了时间的验证。近期也已移植到 Android / macOS / Win32 / POSIX 平台，一并开源。

### MMKV 源起

在微信客户端的日常运营中，时不时就会爆发特殊文字引起系统的 crash，参考文章，文章里面设计的技术方案是在关键代码前后进行计数器的加减，通过检查计数器的异常，来发现引起闪退的异常文字。在会话列表、会话界面等有大量 cell 的地方，希望新加的计时器不会影响滑动性能；另外这些计数器还要永久存储下来——因为闪退随时可能发生。这就需要一个性能非常高的通用 key-value 存储组件，我们考察了 SharedPreferences、NSUserDefaults、SQLite 等常见组件，发现都没能满足如此苛刻的性能要求。考虑到这个防 crash 方案最主要的诉求还是实时写入，而 mmap 内存映射文件刚好满足这种需求，我们尝试通过它来实现一套 key-value 组件。

### MMKV 原理

- **内存准备**
通过 mmap 内存映射文件，提供一段可供随时写入的内存块，App 只管往里面写数据，由操作系统负责将内存回写到文件，不必担心 crash 导致数据丢失。
- **数据组织**
数据序列化方面我们选用 protobuf 协议，pb 在性能和空间占用上都有不错的表现。考虑到我们要提供的是通用 kv 组件，key 可以限定是 string 字符串类型，value 则多种多样（int/bool/double 等）。要做到通用的话，考虑将 value 通过 protobuf 协议序列化成统一的内存块（buffer），然后就可以将这些 KV 对象序列化到内存中。
- **写入优化**
标准 protobuf 不提供增量更新的能力，每次写入都必须全量写入。考虑到主要使用场景是频繁地进行写入更新，我们需要有增量更新的能力：将增量 kv 对象序列化后，直接 append 到内存末尾；这样同一个 key 会有新旧若干份数据，最新的数据在最后；那么只需在程序启动第一次打开 mmkv 时，不断用后读入的 value 替换之前的值，就可以保证数据是最新有效的。
- **空间增长**
使用 append 实现增量更新带来了一个新的问题，就是不断 append 的话，文件大小会增长得不可控。例如同一个 key 不断更新的话，是可能耗尽几百 M 甚至上 G 空间，而事实上整个 kv 文件就这一个 key，不到 1k 空间就存得下。这明显是不可取的。我们需要在性能和空间上做个折中：以内存 pagesize 为单位申请空间，在空间用尽之前都是 append 模式；当 append 到文件末尾时，进行文件重整、key 排重，尝试序列化保存排重结果；排重后空间还是不够用的话，将文件扩大一倍，直到空间足够。
- **数据有效性**
考虑到文件系统、操作系统都有一定的不稳定性，我们另外增加了 crc 校验，对无效数据进行甄别。在 iOS 微信现网环境上，我们观察到有平均约 70万日次的数据校验不通过。


更详细的设计原理参考 MMKV 原理。

### 快速上手

安装引入，推荐使用 Maven：
```
dependencies {
    implementation 'com.tencent:mmkv-static:1.2.7'
}
```  
MMKV 的使用非常简单，所有变更立马生效，无需调用 sync、apply。 在 App 启动时初始化 MMKV，设定 MMKV 的根目录（files/mmkv/），例如在 `Application` 里：
```java
public void onCreate() {
    super.onCreate();

    String rootDir = MMKV.initialize(this);
    System.out.println("mmkv root: " + rootDir);
    //……
}
```
#### CRUD 操作

MMKV 提供一个**全局的实例**，可以直接使用：
```java
import com.tencent.mmkv.MMKV;
...
MMKV kv = MMKV.defaultMMKV();

kv.encode("bool", true);
System.out.println("bool: " + kv.decodeBool("bool"));

kv.encode("int", Integer.MIN_VALUE);
System.out.println("int: " + kv.decodeInt("int"));

kv.encode("long", Long.MAX_VALUE);
System.out.println("long: " + kv.decodeLong("long"));

kv.encode("float", -3.14f);
System.out.println("float: " + kv.decodeFloat("float"));

kv.encode("double", Double.MIN_VALUE);
System.out.println("double: " + kv.decodeDouble("double"));

kv.encode("string", "Hello from mmkv");
System.out.println("string: " + kv.decodeString("string"));

byte[] bytes = {'m', 'm', 'k', 'v'};
kv.encode("bytes", bytes);
System.out.println("bytes: " + new String(kv.decodeBytes("bytes")));
```

#### 删除 & 查询：

```java
MMKV kv = MMKV.defaultMMKV();

kv.removeValueForKey("bool");
System.out.println("bool: " + kv.decodeBool("bool"));

kv.removeValuesForKeys(new String[]{"int", "long"});
System.out.println("allKeys: " + Arrays.toString(kv.allKeys()));

boolean hasBool = kv.containsKey("bool");
```
如果不同业务需要**区别存储**，也可以单独创建自己的实例：
```java
MMKV kv = MMKV.mmkvWithID("MyID");
kv.encode("bool", true);
```
如果业务需要**多进程访问**，那么在初始化的时候加上标志位 `MMKV.MULTI_PROCESS_MODE`：
```java
MMKV kv = MMKV.mmkvWithID("InterProcessKV", MMKV.MULTI_PROCESS_MODE);
kv.encode("bool", true);
```

#### 支持的数据类型

- 支持以下 Java 语言基础类型：
 - boolean、int、long、float、double、byte[]
- 支持以下 Java 类和容器：
  - String、Set<String>
  - 任何实现了Parcelable的类型

#### SharedPreferences 迁移

MMKV 提供了 `importFromSharedPreferences()` 函数，可以比较方便地迁移数据过来。

MMKV 还额外实现了一遍 `SharedPreferences`、`SharedPreferences.Editor` 这两个 interface，在迁移的时候只需两三行代码即可，其他 CRUD 操作代码都不用改。

```java
private void testImportSharedPreferences() {
    //SharedPreferences preferences = getSharedPreferences("myData", MODE_PRIVATE);
    MMKV preferences = MMKV.mmkvWithID("myData");
    // 迁移旧数据
    {
        SharedPreferences old_man = getSharedPreferences("myData", MODE_PRIVATE);
        preferences.importFromSharedPreferences(old_man);
        old_man.edit().clear().commit();
    }
    // 跟以前用法一样
    SharedPreferences.Editor editor = preferences.edit();
    editor.putBoolean("bool", true);
    editor.putInt("int", Integer.MIN_VALUE);
    editor.putLong("long", Long.MAX_VALUE);
    editor.putFloat("float", -3.14f);
    editor.putString("string", "hello, imported");
    HashSet<String> set = new HashSet<String>();
    set.add("W"); set.add("e"); set.add("C"); set.add("h"); set.add("a"); set.add("t");
    editor.putStringSet("string-set", set);
    // 无需调用 commit()
    //editor.commit();
}
```
MMKV 支持多进程访问，更详细的用法参考 Android Tutorial。
