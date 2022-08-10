开发Android应用时，有时候Java层的编码不能满足实现需求，就需要到C/C++实现后生成SO文件，再用`System.loadLibrary()`加载进行调用，这里成为JNI层的实现。常见的场景如：加解密算法，音视频编解码等。在生成SO文件时，需要考虑适配市面上不同手机CPU架构，而生成支持不同平台的SO文件进行兼容。

目前Android共支持七种不同类型的CPU架构，分别是：ARMv5，ARMv7 (从2010年起)，x86 (从2011年起)，MIPS (从2012年起)，ARMv8，MIPS64和x86_64 (从2014年起)。如果你要完美兼容所有类型的手机，理论上是要在的libs目录下放置各个架构平台的SO文件。

这样一来，虽然可以兼容所有机型，但你的项目体积也会变得非常庞大。是否一定需要带入这么多SO文件去兼容呢？答案是否定的。

#### SO（CPU）的兼容

对于CPU来说，不同的架构并不意味着一定互不兼容，根据目前Android共支持七种不同类型的CPU架构，其兼容特点可总结如下：
`armeabi`设备只兼容armeabi；
`armeabi-v7a`设备兼容armeabi-v7a、armeabi；
`arm64-v8a`设备兼容arm64-v8a、armeabi-v7a、armeabi；
`X86`设备兼容X86、armeabi；
`X86_64`设备兼容X86_64、X86、armeabi；
`mips64`设备兼容mips64、mips；
`mips`只兼容mips；
根据以上的兼容总结，我们还可以得到一些规律：

armeabi的SO文件基本上可以说是万金油，它能运行在除了mips和mips64的设备上，但在非armeabi设备上运行性能还是有所损耗；
64位的CPU架构总能向下兼容其对应的32位指令集，如：x86_64兼容X86，arm64-v8a兼容armeabi-v7a，mips64兼容mips；
关于SO的兼容规律就介绍到此，下面谈谈适配工作。

#### SO的适配

从目前移动端CPU市场的份额数据看，ARM架构几乎垄断，所以，除非你的用户很特殊，否则几乎可以不考虑单独编译带入X86、X86_64、mips、mips64架构SO文件。除去这四个架构之后，还要带入armeabi、armeabi-v7a、arm64-v8a这三个不同类型，这对于一个拥有大量SO文件的应用来说，安装包的体积将会增大不少。

针对以上情况，我们可以应用的设备分布和市场情况再进行取舍斟酌，如果你的应用仍有不少armeabi类型的设备，可以考虑只保留armeabi目录下的SO文件（万金油特性）。但是，尽管armeabi可以兼容多种平台，仍有些运算在armeabi-v7a、arm64-v8a去使用armeabi的SO文件时，性能会非常差强人意，所以还是应该用其对应平台架构的SO文件进行运算。注意，这里并不是要带多一整套SO文件到不同的目录下，而是将性能差异比较明显的某个armeabi-v7a、arm64-v8a平台下的SO文件放到armeabi目录，然后通过代码判断设备的CPU类型，再加载其对应架构的SO文件，很多大厂的应用便是这么做的。如微信的lib下虽然只有armeabi一个目录，但目录内的文件仍放着v5、v7a架构的SO文件，用于处理兼容带来的某些性能运算问题。

就目前市场份额而言，绝大部分的设备都已经是armeabi-v7a、arm64-v8a，你也可以考虑只保留armeabi-v7a架构的SO文件，这样能获得更好的性能效果。



参考：
[Android cpu架构兼容so库问题](https://www.jianshu.com/p/83086940a8dd?from=groupmessage)
[NDK 开发中，各种指令集的坑，arm64](https://www.cnblogs.com/xiaoxiaoqingyi/p/7093502.html)
[使用第三方库出现找不到so库UnsatisfiedLinkError错误的原因以及解决方案](https://www.jianshu.com/p/b9a524f24b7e)
