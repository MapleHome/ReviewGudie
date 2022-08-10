## 协程

**是什么？**

- 官方说：协程是轻量级线程
- 协程就是 Kotlin 提供的一套基于线程封装的API，一个比较方便的`线程框架`
- 协程就是切线程
- 挂起就是可以自动切回来的切线程

**特点：**

- 设计初衷是解决并发问题，让`协作式多任务`实现起来更加方便。
- `非阻塞式挂起`：看起来同步的方式写出异步的代码

同一个代码块里进行多次线程切换


```java
fun CoroutionsSope.launch(
    context : CoroutineContext =
    start:CoroutineStart =
    block : suspend CoroutineScope() -> Unit
) : Job
```
示例：
```java
GlobalScope.launch(Dispatchers.Main){
    // 一条线 自动切换执行
    1 -> ioCode()
    2 -> uiCode()
    3 -> ioCode2()
    4 -> uiCode2()
}

suspend fun ioCode(){
    withContext(Dispatchers.IO){
        // ...
    }
}
```











----
