我们回忆一下日常代码的编写: 
1. 抽象业务模型  
2. 根据业务模型编写对应的类  
3. 根据类生成对象用于实际的业务开发  

类是啥? 简单来说就是我们.h文件里面的@interface后面跟的那一串字符.  
对象是啥? [[类 alloc] init]最终的产物就是对象.  

再形象一点, 类就是我们冰淇淋的模具, 总体什么形状是由它决定的, 而对象就是用这个模具产生的冰淇淋.  
最后该模具产生的冰淇淋, 形状必定是相似的, 但是口味可以放入不同的原材料而产生不同的味道.一个冰淇淋模具可以生成若干个冰淇淋.  
换到开发中, 内存中只需要一份类, 对象的话, 生成一个就需要分配一块新的内存空间. 一个类 : 多个对象.    


我们再回忆一下编写类时常编写的内容: 成员变量, 类方法, 实例方法, 协议, 属性等.  
其实总结起来就两东西: 成员变量 + 方法.  

那变量和方法该怎么存放呢?  
首先类是开辟了唯一的一块内存空间(模具只需要一个), 变量每一个对象都是不同的, 相当于不同的冰激凌口味, 所以变量应该存储在各自的内存空间内. 那方法呢? 方法就相当于冰激凌的纹路, 不同原料产生的味道不同, 但是花纹都是一模一样的, 所以直接存在类的内存空间内就可以了, 完全没有必要每个对象单独存一份, 增大开销.  


## 代码分析

上面通过类比简单讲了类的总体结构以后, 我们从源码的层次再来看下其设计与实现.  

OC大多数对象都是继承于NSObject. 

```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
    // 省略其方法
}
```

我们可以看到凡是继承于NSObject的都包含一个isa.   
```
typedef struct objc_class *Class;
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    class_rw_t *data() { 
        return bits.data();
    }
}

struct objc_object {
private:
    isa_t isa;
public:
}
```

我们大致看下源码中设计到的几个数据结构.  

### objc_object与整体闭环

我们看下苹果官方文档对[objc_object](https://developer.apple.com/documentation/objectivec/objc_object)的描述:  
> A pointer to an instance of a class.  
  
简单翻译一下, 指向某个类实例的指针.  

> When you create an instance of a particular class, the allocated memory contains an objc_object data structure, which is directly followed by the data for the instance variables of the class.  

我们根据特定类生成实例的时候, 分配的堆内存会依次存放objc_object,实例变量等等.  

也就是说, 我们通常创建的类的实例, 其内存起始处都有一个objc_object, 它有一个成员变量isa, 按照释义来说, 它是一个指向类实例的指针.  这就很费解了, 我们现在创建的不就是该类的实例吗? 这个指向自己是什么骚操作?  

1. 我们可以看到NSObject源码中的isa的类型是Class, 我们可以大胆猜测我们类的实例中的isa指向的是类.  
2. 源码中objc_class继承于objc_object.说明OC里面的类也是对象.

简而言之, 对象的isa指向类对象.  

objc_class因为继承于objc_object, 所以类对象也有isa, 那类对象的isa又指向什么呢?  

这边我们直接引用网上的图来展示整个闭合的继承回路.  

![OC类闭环](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/object_model.png)

结论1:   
实例对象isa -> 类对象; 类对象isa-> 元类metaclass; 元类metaclass isa-> 根元类root metaclass; 根元类指向本身.  
结论2:  
类对象和元类对象有相同的继承关系.  


第二个结论怎么理解呢? 
上面我们说过 变量是存储在每个实例自己的开辟的内存空间的, 而方法因为可以复用, 所以存在类中即可. 
那类也是一个对象, 那按照同样的逻辑, 类方法应当存在元类中. 按照我们方法调用的规则, 如果没找到就去父类寻找, 而类方法又是存在元类中的, 所以类对象和元类对象应该具有相同的继承关系.  

### 简单成员

Class superclass: 指向父类. 如果该类已经是最顶层的根类(NSObject或NSProxy), 则super_class为NULL.  
cache_t cache: 缓存最近使用的方法. 我们上面说过, 对象的方法是存在类中的, 每次调用查找的话影响效率, cache会缓存最近调用的方法, 下次调用的时候会优先去cache中查找, 如果没有, 再到methodLists中去查找.    


# 引用
[Objective-C Runtime 运行时](http://southpeak.github.io/2014/10/25/objective-c-runtime-1/)  
