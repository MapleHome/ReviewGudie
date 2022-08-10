扔物线
[Kotlin源码里成吨的noinline和crossinline是干嘛的？](https://zhuanlan.zhihu.com/p/224965169)


inline 可以让你用内联——也就是函数内容直插到调用处——的方式来优化代码结构，从而减少函数类型的对象的创建；

noinline 是局部关掉这个优化，来摆脱 inline 带来的「不能把函数类型的参数当对象使用」的限制；

crossinline 是局部加强这个优化，让内联函数里的函数类型的参数可以被间接调用。
