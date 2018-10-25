
# 前言
本篇文章主要围绕iOS内存管理简单展开一下, 主要涉及有MRC,ARC,AutoReleasePool.  

Objective-C在内存管理采用的是引用计数的技术.  
什么是引用计数呢？  
Objective-C的对象可以有多个“拥有者”, 只要有任意一个拥有者还持有对象，那么对象就不能被释放掉.当对象没有任何拥有者以后，那么对象会自动被析构.  
类比下就是一个黑暗的办公室,第一个人进去办公的时候需要灯光,就开灯.陆续大家都进来办公,每一个进来的人就是“拥有者”. 中途任意一个人离开,在办公室还有人("拥有者")的情况下,关灯是要出事的.  
Objective-C为每个对象设置了一个integer即引用计数来标识当前有多少拥有者持有该对象,它会随着拥有者的增多或者减少来相应的加减.当减为0时对象就会被析构.

## MRC(MannulReference Counting)

简单来说,就是开发者手动维护其引用计数.总的来说遵循四个原则,下面会按照四个原则简单展开分析一下.  

1. 自己生成的对象,自己持有.

```
id obj0 = [[NSObeject alloc] init];
id obj1 = [NSObeject new];
```
直白一点就是alloc/new/copy/mutableCopy开头的方法创建的对象,创建后即被持有.

2. 非自己生成的对象，自己也能持有
```
/*
 * 持有非自己生成的对象
 */
id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
[obj retain]; // 自己持有对象
```

3. 自己持有的对象，在使用完毕后，需要自己释放.
```
id obj = [[NSObeject alloc] init]; // 此时持有对象
// do something
[obj release]; // 释放对象
/*
 * 指向对象的指针仍就被保留在obj这个变量中
 * 但对象已经释放，不可访问
 */
```

反之,如果不是自己持有的对象,无需释放.
```
id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
[obj release]; // ~~~此时将运行时crash或编译器报error~~~ 非 ARC下,调用该方法会导致编译器报issues.此操作的行为是未定义的,可能会导致运行时crash或者其它未知行为
```

4. 如果你新建一个对象指针指向对象,那么你应该主动获得其所有权(retain),少数情况(string literal)外.

1) 如果你持有某个对象,你能保证他的引用计数不为0,即未被析构.那你在使用的过程中是安全的.  
2) 如果你没有持有某个对象,那么它是否被析构就没有在你的掌控当中,它只能是临时有效,如果你想把对象作为成员变量临时存起来后续使用,那你必须手动持有他,不然对象可能在你后续使用的时候已经处于被析构的状态从而造成crash.  
 

### AutoReleasePool

四大准则的前三条还是蛮清晰的,但是第四条是不是有点费解?   
我们继续从一些例子展开理解下.  

```
NSString* greeting = [NSString stringWithFormat:@"%@, %@!", @"Hello", @"sailor"];
```
按照准则1, 其创建string的方法不含有alloc/new/copy/mutableCopy.所以我们并没有持有该对象.我们不应该释放不是我们持有的对象.那不是内存泄露了吗？  
NSAutoreleasePool就出场了,stringWithFormat生成的对象会被放入到全局的自动释放池里面,稍后自动释放池会自动对所有加入到池中的对象调用release方法.  
这也会产生一个问题,自动释放池释放对象的时机和runloop有关,这个greeting只是在自动释放池释放对象之前的那段时间临时有效!所以如果不是临时使用,那就需要持有它.

```
- (id) getAObjNotRetain {
    id obj = [[NSObject alloc] init]; // 自己持有对象
    [obj autorelease]; // 取得的对象存在，但自己不持有该对象
    return obj;
}
```
像上面的例子, 对于函数的调用者来说, obj对象不是自己创建的,所以释放的任务不属于自己.但是函数内部更不应该释放了,如果函数返回前释放,那么就是在操作野指针从而crash.这时就需要autorelease, 如果函数调用者需要稍后使用就主动retain持有他,同时释放的任务也归函数调用者,如果只是临时使用,因为其处于自动释放池中,所以下次自动释放池会自动对其发送release消息,从而正常析构它.  

## ARC(Automatic Reference Counting)

可以看到MRC的内存管理方式无趣且易错, 所以苹果后续推出了ARC自动来管理内存.实现方式是在编译时期自动在已有代码中插入合适的内存管理代码以及在 Runtime 做一些优化.  

ARC下内存管理标识符:
* __strong: 默认标识符,只有还有一个强指针指向某个对象，这个对象就会一直存活.
* __weak: 声明这个引用不会保持被引用对象的存活，如果对象没有强引用了，弱引用会被置为 nil
* __unsafe_unretained :  声明这个引用不会保持被引用对象的存活，如果对象没有强引用了，它不会被置为 nil。如果它引用的对象被回收掉了，该指针就变成了野指针
* __autoreleasing: 用于标示使用引用传值的参数（id *），在函数返回时会被自动释放掉

```
Number* __strong num = [[Number alloc] init];
```

ARC大概能帮我们处理大部分内存管理问题,还有些我们自己还需要处理：
1. 循环引用
2. 底层 Core Foundation 对象交互的那部分，底层的 Core Foundation 对象由于不在 ARC 的管理下，所以需要自己维护这些对象的引用计数
```
// 创建一个 CTFontRef 对象
CTFontRef fontRef = CTFontCreateWithName((CFStringRef)@"ArialMT", fontSize, NULL);
// 引用计数加 1
CFRetain(fontRef);
// 引用计数减 1
CFRelease(fontRef);
```


## 引用
[Objective-C 中的内存分配](https://hit-alibaba.github.io/interview/iOS/ObjC-Basic/MM.html)

[An In-depth Look At Manual Memory Management In Objective-C](https://www.tomdalling.com/blog/cocoa/an-in-depth-look-at-manual-memory-management-in-objective-c/)
