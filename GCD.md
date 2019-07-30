## CGD是什么?

[官方文档](https://developer.apple.com/documentation/dispatch) 里面有很清晰的解释.  
简单来说: Grand Central Dispatch. 苹果提供的多核编程的解决方案.  

## GCD有什么优势?

多线程编程自身带有复杂性, 手动管理线程的开启和资源的释放不仅繁琐还易错.  
GCD给我们提供了全新的接口, 我们只需要接触两个简单的概念: 队列 + 任务.底层完全帮我们接管线程的处理,省心省力.  

## CGD如何运作的?

> Dispatch queues are FIFO queues to which your application can submit tasks in the form of block objects. Dispatch queues execute tasks either serially or concurrently. Work submitted to dispatch queues executes on a pool of threads managed by the system. Except for the dispatch queue representing your app's main thread, the system makes no guarantees about which thread it uses to execute a task.  


![](https://raw.githubusercontent.com/HighmoreJx/BlogImage/master/gcd-queues%402x-82965db9.png)


往先进先出的队列提交任务, 队列可同步分发也可异步分发, 其底层维护一个线程池来调度任务.  

## CGD有什么?

简单看下[苹果的文档](https://developer.apple.com/documentation/dispatch).  

### DispatchQueue队列

#### 队列的创建

* 系统提供

```swift
class var main: DispatchQueue
```

主队列我们很熟悉了, 对应到底层就是由主线程分发处理.  
```swift
class func global(qos: DispatchQoS.QoSClass) -> DispatchQueue
```

global队列创建的时候有个qos(quality-of-service)的入参, 代表发送该队列执行任务的优先级.  

quality-of-service:  
userInteractive 表示任务需要被立即执行提供好的体验,用来更新UI,响应事件等.这个等级最好保持小规模.  
userInitiated 任务由UI发起异步执行.适用场景是需要及时结果同时又可以继续交互的时候.  
default  
utility 表示需要长时间运行的任务,伴有用户可见进度指示器.经常会用来做计算,I/O,网络,持续的数据填充等任务.这个任务节能.  
background 表示用户不会察觉的任务,使用它来处理预加载,或者不需要用户交互和对时间不敏感的任务. 
unspecified  

* 自主创建

```swift
convenience init(label: String, qos: DispatchQoS = .unspecified, attributes: DispatchQueue.Attributes = [], autoreleaseFrequency: DispatchQueue.AutoreleaseFrequency = .inherit, target: DispatchQueue? = nil)
```

参数很好的反应了队列的大致面貌.可以直接看[文档](https://developer.apple.com/documentation/dispatch/dispatchqueue/2300059-init).


#### 队列对外接口

```swift
func async(execute: DispatchWorkItem)
Schedules a work item for immediate execution, and returns immediately.

func sync(execute: DispatchWorkItem)
Submits a work item for execution on the current queue and returns after that block finishes executing.

//等等
```

具体有哪些参见[文档](https://developer.apple.com/documentation/dispatch/dispatchqueue)  

## 引用

[深入理解 GCD](https://bestswifter.com/deep-gcd/)
[GCD源码原理分析](https://juejin.im/post/5cbdb4b3f265da039257e1e9)
