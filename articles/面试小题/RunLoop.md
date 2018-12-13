## 前言

RunLoop可以说是面试常客了，这边简单从网上总结下大佬们的文章，详情见文末最下方的引用链接.

大家刚接触编程的时候，一般都是打印hello world.回想一下，控制台输出hello world以后,是不是整个程序就结束了？  
我们后面开发的APP本质上就是复杂的hello world.如果都像打印hello world一样，那APP岂不是打开就闪退？  
这明显不符合我们的要求,这时候你可能会想,用while循环啊...恭喜你,就是这个思路.  
RunLoop最直白的翻译： 跑一个循环。  

但如果单纯跑一个while循环,那将极其耗费资源,所以各个平台产生了不同的事件处理模型.  
总的来说就是为了在闲置时能休眠,有任务时立即被唤醒.而RunLoop就是iOS平台下的事件处理模型.  


## 预备知识

Runloop 源码 Objective-C 版本：https://opensource.apple.com/source/CF/CF-635.19/CFRunLoop.c.auto.html  
Runloop 源码最新的 Swift 版本：https://github.com/apple/swift-corelibs-foundation/blob/master/CoreFoundation/RunLoop.subproj/CFRunLoop.c

### RunLoop与线程

还记得前言说的吗,runloop简单来说就是跑一个while(1)循环.那线程和runloop的关系就比较清晰了.  
一个线程里放两个while(1)是不是吃饱了没事干(注意不是嵌套哈)?所以我们大胆得出一个结论.runloop(while(1))和线程一一对应.   
当然如果你只是想开一个线程随便用一下,用完就销毁,那么runloop是不是太重量了?所以runloop就采用了懒加载的方式,第一次访问的时候创建.  
苹果爸爸怕我们滥用,还设定了一些规则.比如只能在一个线程的内部获取runloop(主线程除外)等.

主线程:  
我们APP能常驻运行肯定是开启runloop作为支撑. 我们在主线程对runloop的应用应该是在系统现有对runloop的事件处理机制下进行自定义处理.  

子线程:
之前提过runloop采用懒加载.是因为不是每个线程都需要重量的runloop来执行任务.比如使用线程执行一些预先设定好的运行时间较长的任务,可能就不需要开启runloop.  
runloop应该是为了 - 想要与线程更多交互的场景准备的  
1)使用input source和其他线程通信   
2)线程中使用timer   
3)使用任何performSelectore系列方法   
4)让线程执行周期性任务  

### mach_msg
系统在某个port收发信息所用的函数.

## RunLoop流程解析

### performTask

### callout_to_observer

### sleep

### 完整流程

## Runloop Mode
