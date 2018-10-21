# 本质
语义+实例变量 + setter/getter = 属性

举个🌰
```
// Person.h
#import <Foundation/Foundation.h>
@interface Person : NSObject
{
    // 实例变量
    NSString *personName;
}

// setter                                   
- (void)setPersonName:(NSString *)newPersonName 

// getter
- (NSString *)personName;
@end  
```
下面是实现文件
```
#import Person.h

@implementation Person

// setter
- (void) setPersonName:(NSString *) newPersonName{
    personName = newPersonName;
}

// getter
- (NSString *) personName{
    return personName;
}
@end
```

上面的代码就是很典型的 变量+setter/getter, 当变量一多的时候,就会有很多这种冗长的代码,Objective-C对此进行了简化.
```
// Person.h

#import <Foundation/Foundation.h>

@interface Person : NSObject
@property(nonatomic,strong) NSString * personName;
@end
```
🤡需要说明几点.
* 默认的实例变量名称就不是personName了,而是_personName.
* Objective-C为我们提供了.语法糖, 让我们可以更优雅的使用setter/getter
* 我们可以用synthesize来重新给实力变量命名
* dynamic可以紧致自动合成方法,由程序猿自己编写

# 语义
## 原子性
### atomic(默认) & nonatomic
对setter和getter方法本身加锁, 让我们操作的值不是一个对象写入而另一个尝试读取它的垃圾值.所以并不是我们理解上的“线程安全”
举个🌰
```
@property (atomic, strong) NSString* stringA;

//thread A
for (int i = 0; i < 100000; i ++) {
    if (i % 2 == 0) {
        self.stringA = @"a very long string";
    }
    else {
        self.stringA = @"string";
    }
    NSLog(@"Thread A: %@\n", self.stringA);
}


//thread B
for (int i = 0; i < 100000; i ++) {
    if (self.stringA.length >= 10) {
        NSString* subStr = [self.stringA substringWithRange:NSMakeRange(0, 10)];
    }
    NSLog(@"Thread B: %@\n", self.stringA);
}
```
上面的🌰就会比较大概率出现崩溃的情况,因为atomic是对setter/getter本身加锁,而我们一般出现线程不安全的情况往往是对属性的使用上.

## 读写语义
### readonly & readwrite(默认)
这个比较好理解, 一个只读, 一个可读可写.
⚠️ 如果你想定义一个属性,对于外部是只读的,但是内部使用是可读可写的, 就可以在.h文件中定义其是只读,在.m实现文件中定义其是可读可写的.

### setter & getter
可以给属性的读写重命名.比如修饰BOOL型变量的时候.
`@property (assign, getter=isEditable) BOOL editable;`
这样当访问属性的时候,就可以用更合适的isEditable来表明变量的含义.

## 内存管理
### strong(默认) & weak & assign
属性分为: 值类型(int, long, bool等类型), 对象类型(声明为指针, 指向某个符合语义的内存区域)
当修饰值类型的时候, 我们就选择使用assign.对象类型就考虑使用strong/weak.
Strong修饰属性,会一直持有该属性.即对象存在的时候, 这个属性指向的对象得一直存在,换句话说属性指向的对象一定得比使用者活得久.所以有时会产生循环引用问题.
Weak修饰属性,只是单纯保留一个使用关系,不影响属性指向对象的生命周期.
### copy
建立一个和指向对象内容相同且引用计数为1的新对象,.NSString等一般用其修饰,.


