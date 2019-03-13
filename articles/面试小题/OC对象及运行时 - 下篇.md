本篇主要讲下应用及延展.  

## Rumtime提供的操作函数

[苹果官方API](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)可以查阅到所有相关操作函数.  
[Objective-C Runtime 运行时之一：类与对象](http://southpeak.github.io/2014/10/25/objective-c-runtime-1/)简要介绍了一些函数的用途, 可以结合看下.   

## 关联对象(Associated Object)与分类

```
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
}; 
```

[Objective-C runtime - 分类与关联对象](http://vanney9.com/2017/06/07/objective-c-runtime-category/)比较清晰的用例子分析了category.  

```
@interface NSObject (vc)
- (void)test;
@end
```
编译结束后, NSObject被编译成objc_class结构体, 而NSObject(vc)编译成category_t结构体.  
attachCategories执行以后,会将category中的所有方法,属性,协议添加到类的class_rw_t中.  

添加任何的分类方法前, NSObject的class_rw_t中只有编译器class_ro_t的baseMethodList, 添加分类后, 会往methods数组中添加一个元素(entsize_list_tt), 后添加的分类在数组中的位置靠前, baseMethodList元素在最后面.  

```
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;
    uint32_t oldCount = array()->count;
    uint32_t newCount = oldCount + addedCount;
    setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
    array()->count = newCount;
    memmove(array()->lists + addedCount, array()->lists, oldCount * sizeof(array()->lists[0]));
    memcpy(array()->lists, addedLists, addedCount * sizeof(array()->lists[0]));
}
```

realloc重新扩展空间.把原来的数组复制到后面,把新数组复制到前面. 因此,如果添加的分类方法和原有类重复,则相当于变相”替换”掉了原有的方法.且最后添加的分类方法最先生效.  


> 对于系统自带的类，才会在runtime时加载分类；但是对于自己创建的类，在编译结束之后就直接将分类的方法添加到baseMethodList中了，就好像没有分类一说  

那如何往分类中添加属性(包含实例变量)呢? 中篇中曾经提到过, ivar是编译期的产物(运行时从无到有产生一个类除外), 不能等到运行时再往里面加ivar.那category是运行时增加的, 那岂不是说分类里面不能加实例变量了?   


关联对象就适用于上面的场景.

![](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/objc-ao-associateobjcect.png)  

关联对象即和对象产生一种关联关系. 
其内部实现如上图所示.  

1. 对象AssociationsManager负责存储所有的关联对象.
2. 存储的数据结构采用AssociationsHashMap. key为对象的地址, value为ObjectAssociationMap.
3. ObjectAssociationMap个人觉得可以理解为关联对象.key为属性名称(void *key),value为ObjcAssociation.  
4. ObjcAssociation就是实际存储的内容,包括_policy内存管理语义和_Value属性的值.  
5. 每一个对象都有一个标记位 has_assoc 指示对象是否含有关联对象.  

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object);
```
理解管理对象的管理方式以后,API也比较明了了,需要传入object(对象的地址), key(属性名称), value(具体的存储内容), policy(内存管理语义)  


## 消息转发

[receiver message]编译期确定向接收者发送message,receiver如何响应message要看具体运行时的转发及处理.  


```
void objc_msgSend(id self, SEL cmd, ...)
```

[Objective-C runtime - 消息](http://vanney9.com/2017/06/08/objective-c-runtime-message/)详细的讲了流程, 这边就简单套下其结论.  


objc_msgSend调用lookupImpOrForward函数查找方法实现.下列步骤如果在任一步骤找到方法实现, 即中止  
1. 在缓存cache中寻找.  
2. 在bits中寻找.  
3. 在superclass中重复1.2步骤.
4. 继续沿着类继承关系重复1.2步骤寻找.  

如果都没找到, 就进入方法决议阶段.  

```
+ (BOOL)resolveInstanceMethod:(SEL)selector
+ (BOOL)resolveClassMethod:(SEL)selector
```

1. 查看当前类是否实现了resolve方法, 没有决议失败.  
2. 存在resolve方法, 执行改方法. 该方法一般动态向当前类添加方法, 执行完resolve后, 重新进入方法查找.  


如果方法决议失败, 则给方法最后一个机会: 消息转发.  

```
- (id)forwardingTargetForSelector:(SEL)selector;
```

执行该函数看能否返回一个新的对象, 该对象可以处理该方法.  

如果无法返回能处理该方法的对象, 就进入完整的消息转发.  
```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
- (void)forwardInvocation:(NSInvocation *)anInvocation
```
重写forwardInvocation,可将anInvocation转发给多个对象来处理消息  

如果都没有处理就doesNotRecognizeSelector 抛出异常.  

![](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/forward_tiny.png)


## method swizzling

动态替换方法的实现, 实现hook功能.  

```
typedef struct method_t *Method;
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
};
```

1. name方法的名称, 唯一标识某个方法.   
2. types标识方法的返回值和参数类型.  
3. imp函数指针, 指向方法的实现.  

OC的方法名是不包含参数类型的.  
```
- (void)viewWillAppear:(BOOL)animated;
- (void)viewWillAppear:(NSString *)string;
```
上面的两个方法在runtime看来是同一个方法.  

method swizzling就是改变方法名name和imp实现的对应关系.  

```
#import <objc/runtime.h>
@implementation UIViewController (Tracking)
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];         
        // When swizzling a class method, use the following:
        // Class class = object_getClass((id)self);
        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        BOOL didAddMethod = class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));
        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}
#pragma mark - Method Swizzling
- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}
@end
```

上诉代码是一个简单的追踪页面显示的应用.  

值得注意的点:   

swizzling的逻辑是在load中实现的.  
[Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)大致描述了两个方法的差异.分类中的load方法不会对主类的load方法造成覆盖.   

swizzling总是在dispatch_once中实现.  






## 引用

[Objective-C runtime - 分类与关联对象](http://vanney9.com/2017/06/07/objective-c-runtime-category/)  

[Objective-C runtime - 消息](http://vanney9.com/2017/06/08/objective-c-runtime-message/)

[iOS 运行时之消息转发机制](http://www.enkichen.com/2017/04/21/ios-message-forwarding/)

[Objective-C Runtime 运行时之四：Method Swizzling](http://southpeak.github.io/2014/11/06/objective-c-runtime-4/)  

[Objective-C Method Swizzling 的最佳实践](http://blog.leichunfeng.com/blog/2015/06/14/objective-c-method-swizzling-best-practice/)  

[Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)
