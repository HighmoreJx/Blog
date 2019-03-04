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

has_cxx_dtor: 










[weak分析](https://github.com/HighmoreJx/Blog/blob/master/articles/%E9%9D%A2%E8%AF%95%E5%B0%8F%E9%A2%98/Objective-C%E5%B1%9E%E6%80%A7%E4%B9%8BWeak.md)文章介绍了一个全局散列表存储引用计数,但其实64位的空间是可以分配一部分来存储引用计数值的. 
