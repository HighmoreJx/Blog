[上篇](https://github.com/HighmoreJx/Blog/blob/master/articles/%E9%9D%A2%E8%AF%95%E5%B0%8F%E9%A2%98/OC%E5%AF%B9%E8%B1%A1%E5%8F%8A%E8%BF%90%E8%A1%8C%E6%97%B6%20-%20%E4%B8%8A%E7%AF%87.md)简单描述了OC对象的继承关系及主要成员.这篇主要介绍下复杂成员的含义.  


## isa_t

```
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

};
```
union联合体.上面的代码是简化后的结果且只代表__arm64__.因为iOS应用为__arm64__架构环境. 

指针类型的大小与CPU位数相关, 一个指针所占用的内存在32位CPU为4个字节, 在64位为8个字节.__arm64__占8个字节,占64位.  

```
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    assert(!cls->instancesRequireRawIsa());
    assert(hasCxxDtor == cls->hasCxxDtor());

    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    if (!nonpointer) {
        isa.cls = cls;
    } else {
        isa_t newisa(0);
         newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
        isa = newisa;
    }
}
```

我们主要看Objc对象的内存分配, 通过上面的代码可以看出isa是采用的联合体的struct, 而不是简单地使用cls.  

我们现在就简单分析下结构体中的成员.  

nonpointer: 0 - raw isa,即当前的联合体只有cls, 没有结构体的部分. 1 - 当前isa不是指针, 代表联合体是结构体部分.  

has_assoc: 对象含有或者曾经含有关联引用，没有关联引用的可以更快地释放内存.  

has_cxx_dtor: 该对象是否有 C++ 或者 Objc 的析构器.  

shiftcls: 类的指针, arm64架构有33位存储类指针.  
```
isa.shiftcls = (uintptr_t)cls >> 3;
```
将当前地址右移三位的主要原因是用于将 Class 指针中无用的后三位清除减小内存的消耗，因为类的指针要按照字节（8 bits）对齐内存，其指针后三位都是没有意义的0.  [从 NSObject 的初始化了解 isa](https://github.com/draveness/analyze/blob/master/contents/objc/%E4%BB%8E%20NSObject%20%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BA%86%E8%A7%A3%20isa.md#shiftcls)  


magic: 判断对象是否初始化完成，在arm64中0x16是调试器判断当前对象是真的对象还是没有初始化的空间.  

weakly_referenced: 对象被指向或者曾经指向一个 ARC 的弱变量，没有弱引用的对象可以更快释放.    

deallocating: 对象是否正在释放内存.  

has_sidetable_rc: 判断该对象的引用计数是否过大，如果过大则需要其他散列表来进行存储.  

extra_rc:  存放该对象的引用计数值减一后的结果。对象的引用计数超过 1，会存在这个这个里面，如果引用计数为 10，extra_rc的值就为 9.  


[weak分析](https://github.com/HighmoreJx/Blog/blob/master/articles/%E9%9D%A2%E8%AF%95%E5%B0%8F%E9%A2%98/Objective-C%E5%B1%9E%E6%80%A7%E4%B9%8BWeak.md)文章介绍了一个全局散列表存储引用计数,isa就是对其进行的优化, extra_rc的19位存储引用计数的值减一, 如果存不下了,就还是走散列表那一套.  

## Tagged Pointer

上面我们已经提到过指针大小在arm64下占64位, 所以其实某些情况直接把指针指向的内容存到指针中也未尝不可.  

![](https://img.halfrost.com/Blog/ArticleImage/23_10.png)

NSNumber、NSDate一类的变量本身的值需要占用的内存大小常常不需要8个字节，拿整数来说，4个字节所能表示的有符号整数就可以达到20多亿（注：2^31=2147483648，另外1位作为符号位)，对于绝大多数情况都是可以处理的.

![](https://img.halfrost.com/Blog/ArticleImage/23_11.png)  

把NSNumber的数据封装到NSNumber的指针中, 就可以节约8个字节.  

特点:  
* 专门存小对象, NSNumber/NSDate和NSString. 
* 指针的值不再是地址, 而是存放真正的内容.  
* 它不是一个对象, 内存不在堆中, 没有isa.不用malloc和free所以创建销毁很快.  
* 若对象指针的最低有效位为奇数，则该指针为Tagged Pointer.  

tagged pointer怎么存放数据的可以看 [iOS Tagged Pointer](https://www.jianshu.com/p/e354f9137ba8)  

## class_data_bits_t

```
struct class_data_bits_t {
    uintptr_t bits;
}

class_rw_t *data() { 
   return bits.data();
}

class_rw_t* data() {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
 }

#define FAST_DATA_MASK          0x00007ffffffffff8UL
```

bits占64位, 其中4-48位代表data.这个data指向class_wt结构体.该结构体包含了类的属性,方法,协议等信息.   

```
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;
};
```

method_array_t, property_array_t, protocol_array_t可以很容易理解到, 方法,属性,协议信息是存放在class_rw_t中的.  

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;
    const uint8_t * ivarLayout;
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```

rw代表readwrite, ro代表readonly. ro代表的是编译期确定的内容, 是只读的. rw代表运行时内容, 是可修改的.  

编译器的结构如下图所示:  
![](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/objc-method-before-realize.png)

接下来会进行resize的操作.  
```
const class_ro_t *ro = (const class_ro_t *)cls->data();
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
rw->ro = ro;
rw->flags = RW_REALIZED|RW_REALIZING;
cls->setData(rw);

```
其内存布局会如下图所示:  
![](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/objc-method-after-realize-class.png)


接下来我们简单看下差异.  

rw和ro都有方法列表, 不过rw是array, 而ro是list.  





## 引用

[神经病院 Objective-C Runtime 入院第一天—— isa 和 Class](https://halfrost.com/objc_runtime_isa_class/)


[](http://vanney9.com/2017/06/05/objective-c-runtime-property-method/)
