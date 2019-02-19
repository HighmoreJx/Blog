# copy
这篇文章主要是看[iOS 中的 Copying](https://joeshang.github.io/),然后小结一下.

## 自问自答

🧐 什么是拷贝?  
😃 : 一模一样的来一份呗.   

🧐 拷贝的场景是啥?  
😃: 我们回到熟悉的抄作业. 每当暑假要完的时候, 我们总会在临近前的最后一天疯狂抄大佬的作业. 这个就是拷贝最好的例子了.    
🧐  一模一样会不会被老师抓?
🤫: 咳咳. 对, 抓紧改几道题的答案. 你的改动会不会影响到大佬的那份? 显然是不会的.    
😶 我头铁, 我一个字都不想改.  
😈: 你是在挑战老师的智商吗? 名字都抄一样的.你这拷贝毫无意义, 结局都是GG.  

👨‍🏫 : 拷贝的主要目的是用于拷贝一份新的数据进行修改, 而不会影响到原来的数据.如果不修改, 拷贝就没有必要. 

## Objective-C 中的拷贝
举个🌰:
```
NSInteger b = 3;
NSInteger a = b;
a = 5;
```
NSInteger是值类型, 上面就是最简单的一次拷贝.

举个🍐:
```
@interface Person : NSObject
@property (nonatomic, assign) int age;
@property (nonatomic, strong) NSMuatableString *name;
@end

Person *person1 = [[Person alloc] init];
Person *person2 = person1;
```
🤥 这是OC对象的拷贝.

```
person1.age = 10;
person2. age = 20;
```

最后发现 ~ person1 和 person2 居然同岁了…  
🧐 我们说过, 拷贝的意图在于产生一份相同的数据, 然后可以任意修改且不影响原始版本. 很明显上面的例子根本不符合条件. 所以根本不是拷贝.  

严肃一点看下官网:  [Object copying](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectCopying.html)  
我们主要关注一句话: 
>  If you receive an object from elsewhere in an application but do not copy it, you share the object with its owner (and perhaps others), who might change the encapsulated contents  

简单来说, 上面的例子只是多了一个持有关系, 根本不是拷贝! 只有两个值类型对象的赋值才完全符合我们对拷贝的认知.

😳 很懵逼, 那OC对象要怎么拷贝? 
🧐 是时候看看[苹果官方文档]([NSCopying - Foundation | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/nscopying?language=objc))了.  

😀 简单来说, 就是要实现NSCoying协议.  
```
#import "Person.h"

@interface Person()<NSCopying>
@end

@implementation Person
- (id)init {
    if (self = [super init]) {
        _name = [[NSMutableString alloc] initWithString:@"mike"];
    }
    return self;
}

- (id)copyWithZone:(NSZone *)zone {
    Person *object = [[[self class] allocWithZone:zone] init];
    object.name = _name;
    object.age = _age;
    return object;
}
@end

Person *person1 = [[Person alloc] init];
Person *person2 = [person1 copy];
person1.age = 10;
person2.age = 20;
```
这个时候通过输出, 我们可以发现两个人的年纪已经不同了.

😃 这完全符合我们对拷贝的定义. 程序从堆中开辟出了完全独立的空间重新构造了person2对象. 我们对person2的操作完全独立与person1.

😈一切真的这么简单吗?
```
[person2.name insertString:@"funny " atIndex:0];
```
灵异的事情发生了, person1也被改了.这就完全不符合拷贝的意图了.  

那么问题出在哪里了呢?  
`object.name = _name;`  
上面说过这个根本不是拷贝, 这是多了一个持有关系而已(多了一个引用).相当于一个盒子有两把钥匙, 盒子里面装的东西变了, 那随便你用哪把钥匙打开盒子, 最后里面的东西肯定是最新放入的.  
指针对象虽然被拷贝了, 但两者还是指向同一块内存空间.  

### 深拷贝与浅拷贝

![object copy](https://joeshang.github.io/images/blog/ios-object-copy.png)

上面是苹果官方文档的一幅图, 图中左侧就很清晰的展示了我们上面所描述的person.name的场景.  
同时我们注意到, 图中右侧还有一副对比图.两幅图分别对应了程序开发的浅复制与深复制.  

浅复制: 指针拷贝, 仅仅拷贝指向对象的指针  
深复制: 内容拷贝, 会拷贝对象本身  



## 基础概念



