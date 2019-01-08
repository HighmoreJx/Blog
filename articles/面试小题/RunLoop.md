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
http://web.mit.edu/darwin/src/modules/xnu/osfmk/man/mach_msg.html  
可以简单将mach_msg理解为多进程之间的一种通信机制,不同的进程可以使用同一个消息队列来交流数据,当使用mach_msg从消息队列里读取数据时,可以在参数设置timeout值,在timeout之前如果没有读到msg,当前线程会一直处于休眠状态.  

## RunLoop流程解析

下面的部分主要是看mrpeak大佬对runloop的总结,但是我更改了部分内容,不知道正确与否,不过我自己的理解是这样,如果有问题欢迎指出.  
```
while (alive) {
  callout_to_observer() //通知外部
  performTask() //执行任务
  sleep() //休眠
}
```
每一次runloop的执行,主要会做上面的三件事情,下面会简单展开一下.

### callout_to_observer

```
if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
```
上面是从源码里面截取下来的片段,通过命名(kCFRunLoopBeforeTimers,kCFRunLoopBeforeSources)可以看出,主要是通知外部,即将(before)处理什么任务.  

DoObservers-Timer: 通知外部即将处理timer    
DoObservers-Source: 通知外部即将处理source    
DoObservers-Activity: 通知外部自己的当前状态  

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

kCFRunLoopEntry: 每次runloop进入时的activity,runloop每一次进入一个mode,就通知一次外部kCFRunLoopEntry,之后会一直以该mode进行,直到当前mode被终止,进而切换到其他mode,并在此通知kCFRunLoopEntry.

kCFRunLoopBeforeTimers, kCFRunLoopBeforeSources: 即将处理timer,source.

kCFRunLoopBeforeWaiting:当前线程即将可能进入睡眠,如果能从内核队列上读出msg则继续进行任务,如果当前队列上没有多余消息,则进入睡眠状态.  
`__CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), poll ? 0 : TIMEOUT_INFINITY);`  

kCFRunLoopAfterWaiting: 线程从睡眠状态中恢复过来,即mach_msg从队列中读出了msg,可以继续执行任务了.这是每一次runloop从idle状态中恢复必调的一个activity,如果想设计一个工具检测runloop的执行周期,这个activity就可作为周期的开始.  

kCFRunLoopExit: 退出,切换mode的时候可能会调用这个activity.  


#### DoObservers-Activity

### performTask
执行task的方式有多种,有些可以被开发者使用,有些则只能被系统使用.  

#### DoBlocks()

这种方式可以被开发者使用, 使用方式比较简单.  
`void CFRunLoopPerformBlock(CFRunLoopRef rl, CFTypeRef mode, void (^block)(void));`  

详细使用方式可[参考文档](https://developer.apple.com/documentation/corefoundation/1542985-cfrunloopperformblock?language=objc)

通过函数签名可以看出,block插入队列的时候,会绑定到某个具体的runloop mode.这个后续会简单讲下runloop mode.  
绑定好以后, runloop在执行的时候, 会通过runloop的循环中的如下代码来执行队列里的所有block.    
`__CFRunLoopDoBlocks(rl, rlm);`  
rlm是CFRunLoopModeRef, 这边也可以看出,只会执行和某个mode相关的所有block.  

#### DoSources0()

RunLoop中有两种source, source0和source1.这个名字取得有点风骚.两个的运行机理有点不同.  
source0有公开的API可以供开发者调用,source1只能供系统调用.source1的实现原理是基于mach_msg函数,通过读取某个port上内核消息队列上的消息来决定执行的任务.  

开发者使用source0:  
1) 创建一个CFRunLoopSourceContext,context里需要传入被执行任务的函数指针作为参数.  
2) 将context作为构造参数传入CFRunLoopSourceCreate创建一个source.  
3) 通过CFRunLoopAddSource将该source绑定到某个runloop mode即可.  

详细使用方式可[参考文档](https://developer.apple.com/documentation/corefoundation/1542679-cfrunloopsourcecreate?language=objc)  

#### DoSources1()

不对开发者开放,系统会使用它来执行一些内部任务,比如渲染UI.  

#### DoTimers
开发者使用NSTimer相关API注册被执行的任务,runloop通过一下API执行相关任务:  
`__CFRunLoopDoTimers(rl, rlm, mach_absolute_time());`  

#### DoMainQueue
开发者调用CGD的API将任务放到main queue中,runloop通过如下API执行被调度的任务:  
`_dispatch_main_queue_callback_4CF(msg);`

### sleep
有任务就执行,没任务就休眠.  

### 完整流程

![runloop流程](https://raw.githubusercontent.com/HighmoreJx/BlogImage/master/rl00.png)

上面的流程图是mrpeak博客上取的,感觉很清晰易懂.下面会结合代码(官网可能会更新)来体会一下这个流程.源代码太多,这边会截取关键的部分.   

```
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
   // 通知 Observers: RunLoop 进入 loop。 
   if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
   result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
   // 通知 Observers: RunLoop 退出 loop
   if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}


static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {

    do {
	__CFPortSet waitSet = rlm->_portSet;
	//通知 Observers: RunLoop 即将触发 Timer 回调。
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

	//执行DoBlocks
	__CFRunLoopDoBlocks(rl, rlm);

	//执行DoSources0
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
	    //如果有source0, 则执行DoBlocks.
            __CFRunLoopDoBlocks(rl, rlm);
	}

	// 如果处理了source0 或者 超时了
        Boolean poll = sourceHandledThisLoop || (0LL == timeout_context->termTSR);

        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
	    //从缓冲区提取消息
            msg = (mach_msg_header_t *)msg_buffer;
	    //从dispat端口读取消息,注意最后一个参数是0.如果没有读到消息是不会进入睡眠期的,会立即返回
	    //这种设计应该是为了保障 dispatch 到 main queue 的代码总是有较高的机会得以运行
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), 0)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }

        didDispatchPortLastTime = false;

    // 如果之前处理了source0消息,那么就不会再observer-beforewaiting了
	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	__CFRunLoopSetSleeping(rl);
	// do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.

        __CFPortSetInsert(dispatchPort, waitSet);

	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
        if (kCFUseCollectableAllocator) {
            objc_clear_stack(0);
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        //  如果消息队列没有消息 且 之前没有处理source0消息,则进入休眠状态
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), poll ? 0 : TIMEOUT_INFINITY);
#elif DEPLOYMENT_TARGET_WINDOWS
        __CFRunLoopWaitForMultipleObjects(waitSet, NULL, poll ? 0 : TIMEOUT_INFINITY, rlm->_msgQMask, &livePort, &windowsMessageReceived);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.

        __CFPortSetRemove(dispatchPort, waitSet);

	rl->_ignoreWakeUps = true;

        // user callouts now OK again
	__CFRunLoopUnsetSleeping(rl);
    // 如果之前处理了source0消息,那么就不会再observer-afterwaiting了
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

        handle_msg:;
	rl->_ignoreWakeUps = true;

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
        mach_port_t livePort = msg ? msg->msgh_local_port : MACH_PORT_NULL;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
        if (windowsMessageReceived) {
            // These Win32 APIs cause a callout, so make sure we're unlocked first and relocked after
            __CFRunLoopModeUnlock(rlm);
	    __CFRunLoopUnlock(rl);

            if (rlm->_msgPump) {
                rlm->_msgPump();
            } else {
                MSG msg;
                if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE | PM_NOYIELD)) {
                    TranslateMessage(&msg);
                    DispatchMessage(&msg);
                }
            }
            
            __CFRunLoopLock(rl);
	    __CFRunLoopModeLock(rlm);
 	    sourceHandledThisLoop = true;
        } else
#endif
        if (MACH_PORT_NULL == livePort) {
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            // do nothing on Mac OS
#if DEPLOYMENT_TARGET_WINDOWS
            // Always reset the wake up port, or risk spinning forever
            ResetEvent(rl->_wakeUpPort);
#endif
        } else if (livePort == rlm->_timerPort) {
#if DEPLOYMENT_TARGET_WINDOWS
            // We use a manual reset timer to ensure that we don't miss timers firing because the run loop did the wakeUpPort this time
            // The only way to reset a timer is to reset the timer using SetWaitableTimer. We don't want it to fire again though, so we set the timeout to a large negative value. The timer may be reset again inside the timer handling code.
            LARGE_INTEGER dueTime;
            dueTime.QuadPart = LONG_MIN;
            SetWaitableTimer(rlm->_timerPort, &dueTime, 0, NULL, NULL, FALSE);
#endif
        // task: 处理timers
	    __CFRunLoopDoTimers(rl, rlm, mach_absolute_time());
        } else if (livePort == dispatchPort) {
        // task: 处理main queue
	    __CFRunLoopModeUnlock(rlm);
	    __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
 	    _dispatch_main_queue_callback_4CF(msg);
#elif DEPLOYMENT_TARGET_WINDOWS
            _dispatch_main_queue_callback_4CF();
#endif
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
	    __CFRunLoopLock(rl);
	    __CFRunLoopModeLock(rlm);
 	    sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
            // 处理source1
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
		mach_msg_header_t *reply = NULL;
		sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
		if (NULL != reply) {
		    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
		    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
		}
#elif DEPLOYMENT_TARGET_WINDOWS
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
#endif
	    }
        } 
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
	__CFRunLoopDoBlocks(rl, rlm);

	if (sourceHandledThisLoop && stopAfterHandle) {
	    retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < (int64_t)mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut;
	} else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
	    retVal = kCFRunLoopRunStopped;
	} else if (rlm->_stopped) {
	    rlm->_stopped = false;
	    retVal = kCFRunLoopRunStopped;
	} else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
	    retVal = kCFRunLoopRunFinished;
	}
    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}

```

## Runloop Mode

文章前面介绍API等的时候,都反反复复提到了一个概念: Mode.  
我们接下来会围绕Mode来展开.  

### Mode的总览

![Mode与RunLoop](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_0.png)

```
struct __CFRunLoopMode {
    ...
    CFStringRef _name;
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    CFIndex _observerMask;
	...
};

struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};
```

一个RunLoop包含若干个Mode.每个Mode包含若干个Source/Timer/Observer. 每次调用RunLoop的主函数时,只能指定其中一个Mode,即_currentMode.如果要切换Mode,就只能退出Loop,重新指定一个Mode后再进入.  

我们注意到RunLoop中出现了items, 那item又是什么呢? 我们之前提到的Source/Timer/Observer实际上就是model item.一个item可以被同时加入多个mode.但一个item被重复加入同一个mode是不会有效果的.如果一个mode中一个item都没有,则RunLoop会直接退出,不进入循环. 

以doTimers为例
```
static Boolean __CFRunLoopDoTimers(CFRunLoopRef rl, CFRunLoopModeRef rlm, int64_t limitTSR) {	/* DOES CALLOUT */
    for (CFIndex idx = 0, cnt = rlm->_timers ? CFArrayGetCount(rlm->_timers) : 0; idx < cnt; idx++) {
		...
    }
    return timerHandled;
}
```
mode定义的数据结构里,包含了执行任务和通知外部所需要的信息. 上面的代码执行doTimer时,就只需要将 _timers 遍历并 invoke 即可.


### Mode的分类

简单来说, mode分为两类: common mode和private mode.  

公开的mode: http://iphonedevwiki.net/index.php/CFRunLoop

我们熟悉的kCFRunLoopDefaultMode 和 UITrackingRunLoopMode就是common mode.kCFRunLoopDefaultMode是默认模式下runloop使用的mode,scrollview滑动时切换到UITrackingRunLoopMode.  

系统定义了一些private mode.比如 UIInitializationRunLoopMode 和 GSEventReceiveRunLoopMode.

kCFRunLoopCommonModes比较特殊一些, 是一个占位用的Mod.既然是占位符,为什么会有下面的写法呢?  
```
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```
我们回过头看RunLoop的结构,有两个成员变量_commonModes,_commonModeItems.commonModes可以理解为存放所有common型的Mode的集合,每当RunLoop的内容发生改变时,_commonModeItems里的Source/Observer/Timer同步到_commonModes存放的Mode中.上面的代码段简单来说就是把timer加入到_commonModeItems的集合里面.


#### Mode切换的经典问题-NSTimer

列表滑动时, 定时器不会被触发?

NSTimer默认只会调度到kCFRunLoopDefaultMode 模式下,当scrollview滑动时,runloop会进入UITrackingRunLoopMode.那么在doTimer时,遍历当前mode的timers数组时,就无法找到.自然也就不会触发NSTimer的任务.

怎么解决呢?  

不能执行的原因是滑动时的Mode不含有timer, 那我们timer加入到UITrackingRunLoopMode不就可以了?
```
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
[[NSRunLoop mainRunLoop] addTimer:timer forMode: UITrackingRunLoopMode];
```

不过连续写两句类似的代码是有点奇怪哦.我们回想下NSDefaultRunLoopMode和UITrackingRunLoopMode都是common mode.之前我们已经提到过了,commonModeItems集合里的元素会自动被加入到common型的Mode里,那我们直接把其加入到commonModeItems不就可以了?
```
[[NSRunLoop mainRunLoop] addTimer:timer forMode: NSRunLoopCommonModes]; 
```

这样就支持所有的情况了吗?很显然不是的, 还有很多private Mode.如果runLoop处于任何一种private Mode下,那定时器还是不会被触发的.但是一般情况已经满足了,如果真的对精度有那么高要求,相比也不会用NSTimer.

还有GCD定时器等方法就不介绍了.  



