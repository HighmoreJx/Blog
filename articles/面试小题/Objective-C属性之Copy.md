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
@property (nonatomic, copy) NSString *name;
@end

Person *person1 = [[Person alloc] init];
Person *person2 = person1;
```
🤥 这是OC对象的拷贝.

```
person1.name = @"one";
person2.name = @"two";
```

最后发现 ~ person1 和 person2 居然重名了…  
🧐 我们说过, 拷贝的意图在于产生一份相同的数据, 然后可以任意修改且不影响原始版本. 很明显上面的例子根本不符合条件. 所以根本不是拷贝.  

严肃一点看下官网:  [Object copying](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectCopying.html)  
我们主要关注一句话: 
>  If you receive an object from elsewhere in an application but do not copy it, you share the object with its owner (and perhaps others), who might change the encapsulated contents  

简单来说, 上面的例子只是多了一个持有关系, 根本不是拷贝! 只有两个值类型对象的赋值才完全符合我们对拷贝的认知.


## 基础概念

浅复制: 指针拷贝, 仅仅拷贝指向对象的指针  
深复制: 内容拷贝, 会拷贝对象本身  

