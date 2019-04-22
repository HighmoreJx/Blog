## 前言

* 命令行程序一般执行完代码, 打印出结果以后就结束(hello world).而GUI程序常驻内存, 采用EventLoop保证主线程随时处理事件且不退出.  
* APP启动时, 创建单例的UIApplication对象, 该对象负责处理和分发系统发送到事件队列中的事件.  

问题:  
1. 屏幕上那么多元素控件, 系统是怎么知道我们点击的是谁?  
2. 点击按钮和普通的视图有什么不同?  
3. 手势和触摸的区分逻辑.  


事件队列中的数据源大致分为以下三类:  
1. UIControl Actions - action / target那一套.  
2. User Events - 触屏, 摇晃等等.  
3. System Events - 低内存, 旋转机身等等.  

## UIControl Actions

我们点击按钮, 滑动滑块, 点击开关都属于UIControl Actions.我们查看文档可以发现, 他们的父类都是[UIControl](https://developer.apple.com/documentation/uikit/uicontrol).它也是在UIView的基础上+一些基础的实现和一些通用的接口.  

我们主要关注的是The Target-Action Mechanism.  

```
func addTarget(_ target: Any?, action: Selector, for controlEvents: UIControl.Event)
func removeTarget(_ target: Any?, action: Selector?, for controlEvents: UIControl.Event)
```
1. UIControl存储一个可变数组_targetActions用于存放私有类UIControlTargetAction.
2. UIControlTargetAction包含target, action, eventMask等基本要素. 
3. remove和add操作的都是可变数组.  
4. UIControlTargetAction的target是弱引用持有.否则会造成循环引用.  

流程:  
1. 对应事件发生(sendActions或控件交互触发), UIControl调用sendAction:to:forEvent:来将消息转发到UIApplication对象上.  
2. UIApplication调用sendAction:to:fromSender:forEvent:将消息转发到指定的target上.  
3. 如果的target为nil, 则会沿着响应链来搜寻想处理这个action的对象(view->superview->controller->uiwindow->uiapplication->appdelegate).如果没有找到就丢弃, 这种特性就是我们常说的nil-targeted actions.  

关于nil-targeted actions在一定程度上可以用来解耦某些东西, 可以充分利用响应链来实现我们自己的数据传递.但是和通知类似, 也伴随风险, 使用时要考虑是否符合自身场景.详细例子可以参考文末资料.  

### 响应链

上面我们说到了一个概念叫响应链, 那具体是什么呢?  

UIResponder
```
- (BOOL)isFirstResponder
- (BOOL)becomeFirstResponder
- (BOOL)canBecomeFirstResponder
- (BOOL)resignFirstResponder
- (BOOL)canResignFirstResponder
```
响应者: UIResponder或者其子类的实力对象, 用来接收和响应事件. 
响应链: 即按照视图层级结构链接所有的响应者而最终形成的链式关系.  

![](https://raw.githubusercontent.com/HighmoreJx/BlogImage/master/respond-chain.png)


1. UIResponder的nextResponder不保存或设置下一个响应者, 方法默认实现是nil. 子类的实现必须重写这个方法来设置下一个响应者.  
2. UIView实现返回管理它的UIViewController或者其父视图. UIViewController返回它的视图的父视图. UIWindow返回app对象.UIApplication实现返回nil. 这也是响应链的构成原理及顺序,可以通过更改对应方法来满足自定义需求, 也代表响应链是在构造视图结构时生成的.  
3. 响应对象只能在当前响应者能放弃第一响应者且自身能成为第一响应者才可成为第一响应者.  
4. 必须处于当前视图层次结构

## User Events

事件只要来源于与用户的交互.大体上分为三类: 触摸事件, 运动事件, 远程控制事件. 

### 触摸事件

[iOS触摸事件的流动](https://shellhue.github.io/2017/03/04/FlowOfUITouch/)这篇文章绘制了从点击屏幕到响应的流程.我们接下来主要沿着这个流程简单分析下.  

![](https://raw.githubusercontent.com/HighmoreJx/BlogImage/master/uitouchflow.png)


#### 系统响应

1. 触摸屏幕, 对应事件传递给IOKit(用来操作硬件和驱动的框架).  
2. IOKit.framework封装整个触摸事件为IOHIDEvent, 通过mach port转发给SpringBoard.app.  

#### SpringBoard.app处理

SpringBoard.app是一个系统进程, 可理解为桌面系统, 统一管理和分发系统接收到的触摸事件.  

1. SpringBoard.app的主线程runloop收到IOKit.framework转发来的消息苏醒, 触发mach port的source1回调__IOHIDEventSystemClientQueueCallback().  
2. 如果SpringBorad.app监测到前台有APP, 通过mach port转发给它. 如果监测到前台没有APP,则SpringBoard.app进入App内部响应阶段,触发自身主线程runloop的Source0时间源的回调.


#### APP内部响应

1. APP主线程Runloop收到SpringBoard.app消息苏醒, 触发mach port的source1回调__IOHIDEventSystemClientQueueCallback().  
2. Source1回调内部,触发Source0回调__UIApplicationHandleEventQueue().  
3. Source0回调内部,封装IOHIDEvent为UIEvent.  
4. Source0回调内部,调用UIApplication的sendEvent:方法,将UIEvent传给UIWindow.  

和底层相关的流程我们可以大致了解下就可以了.但是到UIApplication我们就需要熟悉一些.  

UIEvent被放到事件队列中, UIApplication按照先进先出的顺序读取并分发队列中的事件, 除触摸事件以外的事件都会被分发给第一响应者处理, 如果第一响应者无法处理, 则根据响应链进行传递.(to do)

(1) UIWindow利用hit-testing寻找最合适处理该事件的对象.  

* hit-testing是什么? 

检测触摸事件是否发生在视图的边界内. 使用的搜索方法是逆前序深度遍历, 即当触摸事件发生在多个视图的重叠部分时,根据算法将最先检测最右子树的最深视图,而该视图位于界面最前端. 这样可以省去部分搜索操作.  

```
- (UIView *)hitTest:(CGPoint)point 
          withEvent:(UIEvent *)event;
- (BOOL)pointInside:(CGPoint)point 
          withEvent:(UIEvent *)event;
```
下面是一段实现的代码猜测, 来源于引用资料中.  
```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (!self.isUserInteractionEnabled || self.isHidden || self.alpha <= 0.01) {
        return nil;
    }
    if ([self pointInside:point withEvent:event]) {
        for (UIView *subview in [self.subviews reverseObjectEnumerator]) {
            CGPoint convertedPoint = [subview convertPoint:point fromView:self];
            UIView *hitTestView = [subview hitTest:convertedPoint withEvent:event];
            if (hitTestView) {
                return hitTestView;
            }
        }
        return self;
    }
    return nil;
}
```

即首先视图得允许接收touch, 然后对subview进行逆序遍历.当无子视图且触摸点在当前view的范围内,则返回self,找到hit-view.  

视图不允许接收touch:  
1) 不允许交互的视图：userInteractionEnabled = NO
2) 隐藏的视图：hidden = YES  
3) 透明度alpha<0.01的视图  

(2) hit-testing找到了合适的对象, 沿着该对象所在响应链,可以处理的响应者都可响应(截断后停止)

1) UIApplication先将事件传给hit-testing找到的对象.  
   UIApplication(sendEvent)->window(sendEvent)->view
2) 最佳响应者(hit-testing找到的view)优先处理, 如果没有截断, 事件通过响应链传递到下一个响应者.依次类推.  

```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event;
```

最佳响应链之前已经说过了.大致流程如下:  
* hit-testing找到的view  
* view的superview  
* 如果已经到控制器的rootview还不能处理touch event 则viewcontroller  
* 如果控制器已经是root controller则window  
* application  
* app delegate


响应者对于接收到的事件有三种操作:  
1. 不拦截, 默认操作, 事件会自动沿着响应链传递.  
2. 拦截, 不再往下分发事件.    
3. 拦截, 继续往下分发.  

#### 手势处理

手势处理是Apple提供的更高级的事件处理技术, 可以让我们更细致的区分用户的操作手势从而达到更好的用户体验.  

[UIGestureRecognizer](https://developer.apple.com/documentation/uikit/uigesturerecognizer)为手势处理基类.可以去官方看下文档了解下.  
* UITouch: 一个手指第一次点击屏幕,就会生成一个UITouch对象,到手指离开时销毁.当我们有多个手指触摸屏幕时,会生成多个UITouch对象.  
* UIEvent: 一个触摸事件定义为第一个手指开始触摸屏幕到最后一个手指离开屏幕. 一个UIEvent对象实际上对应多个UITouch对象.  

UIEvent: 包含window, view, gesture信息.UIApplication可以直接解析Event中相应元素, 然后让其相应event.  

![](https://raw.githubusercontent.com/HighmoreJx/BlogImage/master/gesture-state.png)


```
//UIGestureRecognizer(UIGestureRecognizerProtected)
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
```
手势在上面四个方法对手势的State做更改.  

手势: 离散型(点按tap, 轻扫swipe), 持续型(其余)
1) 离散型:
    识别成功: Possible->Recognized
    识别失败: Possible->Failed
2) 持续型:
    完整识别: Possible —> Began —> [Changed] —> Ended
    不完整识别: Possible —> Began —> [Changed] —> Cancel


// to to example


1. 手势识别比UIResponder具有更高的事件响应优先级  

```
A window delivers touch events to a gesture recognizer before it delivers them to the hit-tested view attached to the gesture recognizer. Generally, if a gesture recognizer analyzes the stream of touches in a multi-touch sequence and doesn’t recognize its gesture, the view receives the full complement of touches. If a gesture recognizer recognizes its gesture, the remaining touches for the view are cancelled.The usual sequence of actions in gesture recognition follows a path determined by default values of the cancelsTouchesInView, delaysTouchesBegan, delaysTouchesEnded properties.
```
事件会优先传递给手势识别器, 然后再传递给hit-tested视图. 如果手势识别器没有识别出手势, 则视图会进行完整的respond流程, 如果识别了, 识别以后的触摸事件都会被cancel掉.  


cancelsTouchesInView: 默认为yes, 即手势识别器成功识别手势后, 通知Application取消响应链对事件的响应.如果设置为no, 识别完手势后仍然不影响事件的传递.  
delaysTouchesBegan: 默认为no, 手势识别期间事件会传递给hit-tested的view.设置成yes, 识别期间截断事件.  
delaysTouchesEnded: 默认为yes, 手势识别失败, 如果触摸结束, 会延迟(0.15s)再调用响应者的touchesEnd. 如果是no, 则不会延迟, 马上调用.  


2. UIControl的优先级

control-action上面已经提过了,我们知道最终是UIControl调用sendAction发送给application然后再转发到对应target上.  
那control一开始又是在什么时机调用sendAction的呢?  
```
- (BOOL)beginTrackingWithTouch:(UITouch *)touch withEvent:(nullable UIEvent *)event;
- (BOOL)continueTrackingWithTouch:(UITouch *)touch withEvent:(nullable UIEvent *)event;
- (void)endTrackingWithTouch:(nullable UITouch *)touch withEvent:(nullable UIEvent *)event;
- (void)cancelTrackingWithEvent:(nullable UIEvent *)event;
```
UIControl本身也集成UIResponder, 所以UIResponder的四个touch方法它也同样拥有.那这8个方法到底有什么关联呢?  

1) 职能: 开始, 滑动, 结束, 取消.  
2) UIControl的Tracking系列方法是在touch系列方法内调用的.  

那control的优先级又如何呢? 

> In iOS 6.0 and later, default control actions prevent overlapping gesture recognizer behavior. For example, the default action for a button is a single tap. If you have a single tap gesture recognizer attached to a button’s parent view, and the user taps the button, then the button’s action method receives the touch event instead of the gesture recognizer.This applies only to gesture recognition that overlaps the default action for a control, which includes:
A single finger single tap on a UIButton, UISwitch, UIStepper, UISegmentedControl, and UIPageControl.
A single finger swipe on the knob of a UISlider, in a direction parallel to the slider.
A single finger pan gesture on the knob of a UISwitch, in a direction parallel to the switch.

1) UIResponder和UIControl的优先级遵循响应链层级, 即优先级是相同的.  
2) 默认的control actions会阻止与该action操作相同的手势识别.  

### 其他user事件

移动事件
```
// 移动事件开始
- (void)motionBegan:(UIEventSubtype)motion withEvent:(UIEvent *)event
// 移动事件结束
- (void)motionEnded:(UIEventSubtype)motion withEvent:(UIEvent *)event
// 取消移动事件
- (void)motionCancelled:(UIEventSubtype)motion withEvent:(UIEvent *)event
```

远程控制事件
```
- (void)remoteControlReceivedWithEvent:(UIEvent *)event
```

还有一些其他的, 但是就不继续了, 主要还是理解触摸事件.  
普通的用户事件只是简单的寻找第一响应者, 然后沿着响应链继续. 相比于触摸事件就少了很多步骤, 比如hit-testing.  


## System Events

低内存, 旋转等等.  

系统将事件发送到应用程序单例，这些系统相关事件将由应用程序单例接收并分派给App委托。 应用程序代理将依次接收事件并处理它们.  

## 引用

[UIKit: UIControl](http://southpeak.github.io/2015/12/13/cocoa-uikit-uicontrol/)  

[iOS触摸事件处理](https://www.jianshu.com/p/f73b4bdc73c7)

[Understanding cocoa and cocoa touch responder chain](https://medium.com/ios-os-x-development/understanding-cocoa-and-cocoa-touch-responder-chain-12fe558ebe97)

[Event Delivery on iOS](https://medium.com/bpxl-craft/event-delivery-on-ios-part-1-8e68b3a3f423)

[iOS事件处理之Hit-Testing](https://zhongwuzw.github.io/2016/09/12/iOS%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86%E4%B9%8BHit-Testing/)
