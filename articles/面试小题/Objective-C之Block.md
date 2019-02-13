## 前言

> [闭包](https://zh.wikipedia.org/wiki/%E9%97%AD%E5%8C%85_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6)): 又称词法闭包和函数闭包,是引用了自由变量的函数.这个被引用的自由变量将和这个函数一同存在,即使已经离开了创造它的环境也不例外.所以,有另一种说法认为闭包是函数和其相关的引用环境组合而成的实体.  

block就是闭包在Objective-C中的实现.本篇文章主要会解释以下几个问题:  
1. block的结构是什么?
2. block是如何捕获变量的?
3. __block修饰的变量和普通的变量有什么区别?
4. block的循环引用是怎么造成的?

## 预备工作

### 研究工具

clang.命令行执行以下命令 clang -rewrite-objc block.c block.c就是你需要翻译的文件名. 如果是main.m. 替换掉就好了.

### C语言变量类型

* 自动变量
* 函数参数
* 静态变量
* 静态全局变量
* 全局变量

### 内存布局

![内存布局](https://raw.githubusercontent.com/HighmoreXu/BlogImage/master/images/memory_structure.jpg)

## Block结构

### 源码

[官网源码](https://opensource.apple.com/source/libclosure/libclosure-73/)是73版本的,有更新的话可以修改对应路径就可以了.

Block的定义在Block_private.h中.  
```c

typedef void(*BlockCopyFunction)(void *, const void *);
typedef void(*BlockDisposeFunction)(const void *);
typedef void(*BlockInvokeFunction)(void *, ...);
typedef void(*BlockByrefKeepFunction)(struct Block_byref*, struct Block_byref*);
typedef void(*BlockByrefDestroyFunction)(struct Block_byref *);

// Values for Block_layout->flags to describe block objects
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler
};

#define BLOCK_DESCRIPTOR_1 1
struct Block_descriptor_1 {
    uintptr_t reserved;
    uintptr_t size;
};

#define BLOCK_DESCRIPTOR_2 1
struct Block_descriptor_2 {
    // requires BLOCK_HAS_COPY_DISPOSE
    BlockCopyFunction copy;
    BlockDisposeFunction dispose;
};

#define BLOCK_DESCRIPTOR_3 1
struct Block_descriptor_3 {
    // requires BLOCK_HAS_SIGNATURE
    const char *signature;
    const char *layout;     // contents depend on BLOCK_HAS_EXTENDED_LAYOUT
};

struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved;
    BlockInvokeFunction invoke;
    struct Block_descriptor_1 *descriptor;
    // imported variables
};
```

简单看下Block翻译成C代码的结构 - Block_layout.
* isa -  很熟悉了,OC里面每个对象都有该指针,指向类
* flags - 源码上定义的枚举
* reserved - 保留信息
* invoke - 函数指针,指向block具体的执行函数
* descriptor - block附加描述信息.主要保存了内存size,copy和dispose函数指针,签名,layout等信息.默认情况下block_layout的descriptor的类型是Block_descriptor_1.当捕获到不同类型变量或者没用到外部变量时,编译器会改变结构体结构,按需设置类型.
*  imported variables - 捕获到的外部变量.

### Block类型

下面我们会分别介绍Block的几种类型,每种类型会结合实际代码及工具翻译的C代码来帮助理解.

#### NSConcreteGlobalBlock

```Objective-C
#import <Foundation/Foundation.h>
int main(int argc, const char * argv[]) {
    void(^myBlock)(void) = ^{};
    NSLog(@"%@", myBlock);
    myBlock();
    return 0;
}
```
输出  
`<__NSGlobalBlock__: 0x100001058>  `

转换C代码
```C

我们定位到main函数可以看到如下代码
int main(int argc, const char * argv[]) {
    // void(^myBlock)(void) = ^{};
    void(*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

    // NSLog(@"%@", myBlock);
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_07_p4zsgcc12dsbg_1gbhy9km5m0000gn_T_main_5c5bdd_mi_0, myBlock);

    //myBlock();
    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);

    return 0;
}
```
看上去比较烦躁,但一一对应下来还是比较清晰的.  
首先第一句代码牵涉到了一些数据结构,我们可以在转换后的代码找到.  
```C
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};


struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

__main_block_impl_0就是我们最后生成的Block数据结构,他的构造函数分别接受fp, desc, flags三个参数.
* fp - 函数指针,指向block具体的执行函数
* desc - block附加描述信息,这里主要有内存大小size,和保留信息reserved
* flags - 源码上定义的枚举,上面介绍block结构时提到过.

细心的同学可以发现, 我们控制台直接打印出来的是globalBlock,但是转换成C代码以后,isa又赋值的_NSConcreteStackBlock.使用clang改写和LLVM具体实现内容实际是不同的,不用太纠结,以LLVM为准.  

#### NSConcreteStackBlock

```Objective-C
int main(int argc, const char * argv[]) {
    int i = 5;
    NSLog(@"%@", ^{NSLog(@"********%d", i);});
    return 0;
}
```
输出  
`<__NSStackBlock__: 0x7ffeefbff568>`
转换成C代码  
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int i;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _i, int flags=0) : i(_i) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int i = __cself->i; // bound by copy
NSLog((NSString *)&__NSConstantStringImpl__var_folders_07_p4zsgcc12dsbg_1gbhy9km5m0000gn_T_main_860511_mi_1, i);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    int i = 5;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_07_p4zsgcc12dsbg_1gbhy9km5m0000gn_T_main_860511_mi_0, ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, i)));
    return 0;
}
```

🤥转换后的代码也相对简单, 我们主要看下和上面代码段的差异.
1. llvm打印出block的类型变为了__NSStackBlock__. 
2. 我们可以看到在构造__main_block_impl_0时, i变量是以值拷贝的形式来构造结构体的.所以我们可以得出block内部是无法改变外部环境的自动变量的.所能改变的只是拷贝的副本.

#### NSConcreteMallocBlock
```Objective-C
int main(int argc, const char * argv[]) {
    __block int i = 5;
    void(^myBlock)(void) = ^{NSLog(@"********%d", i++);};
    myBlock();
    NSLog(@"%@", myBlock);
    NSLog(@"********%d", i);
    return 0;
}
```
输出
`<__NSMallocBlock__: 0x100615b80>`
转换成c代码
```
int main(int argc, const char * argv[]) {
    __attribute__((__blocks__(byref))) __Block_byref_i_0 i = {(void*)0,(__Block_byref_i_0 *)&i, 0, sizeof(__Block_byref_i_0), 5};
    void(*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_i_0 *)&i, 570425344));
    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_07_p4zsgcc12dsbg_1gbhy9km5m0000gn_T_main_1ead2a_mi_1, myBlock);
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_07_p4zsgcc12dsbg_1gbhy9km5m0000gn_T_main_1ead2a_mi_2, (i.__forwarding->i));
    return 0;
}
```

😱上面转换的代码就比较多了.不要被吓到了,我们挨着分析一下·
`__block int i = 5;`
对照main函数被转换成了
`__attribute__((__blocks__(byref))) __Block_byref_i_0 i = {(void*)0,(__Block_byref_i_0 *)&i, 0, sizeof(__Block_byref_i_0), 5};`
对应到源码中找到结构体
```
struct __Block_byref_i_0 {
  void *__isa; //isa指针
__Block_byref_i_0 *__forwarding; //指向自身类型的__forwarding指针
 int __flags; //标记flag
 int __size; // 大小
 int i; //变量值(名字和变量名同名)
};
```
🤔️上面我们提到过, 单纯的自动变量是以值拷贝的形式被拷贝到block结构体中的,所以block内部修改捕获到的变量是不会影响外部的自动变量.  
而我们知道,用__block修饰的变量是可以在block内部中被修改的,那又是如何做到的呢?我们接着往下看. 
`void(*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_i_0 *)&i, 570425344));`

继续在源码中查看对应结构体.
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_i_0 *i; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_i_0 *_i, int flags=0) : i(_i->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_i_0 *i = __cself->i; // bound by ref
NSLog((NSString *)&__NSConstantStringImpl__var_folders_07_p4zsgcc12dsbg_1gbhy9km5m0000gn_T_main_1ead2a_mi_0, (i->__forwarding->i)++);}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->i, (void*)src->i, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->i, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
```

🤔️是有点费解哦....我们慢慢来...

👨‍🌾block类型变为了NSConcreteMallocBlock.相比于NSConcreteStackBlock,这边只是多了一个block的赋值操作.block的类型就由栈变为了堆.其转换规则大致如下(ARC):
1. 手动调用copy
2. Block是函数的返回值
3. Block被强引用,Block被赋值给__strong或者id类型
4. 调用系统API入参中含有usingBlock的方法  

以上情况,系统都会默认调用copy方法把Block复制.而MRC的话只有显示调用copy,否则block只是相当于NSConcreteStackBlock

👨‍🌾 block_desc多了两个辅助函数copy和dispose.
我们以copy为例分析.最终调用代码:
`_Block_object_assign((void*)&dst->i, (void*)src->i, 8/*BLOCK_FIELD_IS_BYREF*/);`
详细实现:
```
void _Block_object_assign(void *destAddr, const void *object, const int flags) {
    //printf("_Block_object_assign(*%p, %p, %x)\n", destAddr, object, flags);
    if ((flags & BLOCK_BYREF_CALLER) == BLOCK_BYREF_CALLER) {
        if ((flags & BLOCK_FIELD_IS_WEAK) == BLOCK_FIELD_IS_WEAK) {
            _Block_assign_weak(object, destAddr);
        }
        else {
            // do *not* retain or *copy* __block variables whatever they are
            _Block_assign((void *)object, destAddr);
        }
    }
    else if ((flags & BLOCK_FIELD_IS_BYREF) == BLOCK_FIELD_IS_BYREF)  {
        // copying a __block reference from the stack Block to the heap
        // flags will indicate if it holds a __weak reference and needs a special isa
        _Block_byref_assign_copy(destAddr, object, flags);
    }
    // (this test must be before next one)
    else if ((flags & BLOCK_FIELD_IS_BLOCK) == BLOCK_FIELD_IS_BLOCK) {
        // copying a Block declared variable from the stack Block to the heap
        _Block_assign(_Block_copy_internal(object, flags), destAddr);
    }
    // (this test must be after previous one)
    else if ((flags & BLOCK_FIELD_IS_OBJECT) == BLOCK_FIELD_IS_OBJECT) {
        //printf("retaining object at %p\n", object);
        _Block_retain_object(object);
        //printf("done retaining object at %p\n", object);
        _Block_assign((void *)object, destAddr);
    }
}


static void _Block_byref_assign_copy(void *dest, const void *arg, const int flags) {
    struct Block_byref **destp = (struct Block_byref **)dest;
    struct Block_byref *src = (struct Block_byref *)arg;

    //printf("_Block_byref_assign_copy called, byref destp %p, src %p, flags %x\n", destp, src, flags);
    //printf("src dump: %s\n", _Block_byref_dump(src));
    if (src->forwarding->flags & BLOCK_IS_GC) {
        ;   // don't need to do any more work
    }
    else if ((src->forwarding->flags & BLOCK_REFCOUNT_MASK) == 0) {
        //printf("making copy\n");
        // src points to stack
        bool isWeak = ((flags & (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK)) == (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK));
        // if its weak ask for an object (only matters under GC)
        struct Block_byref *copy = (struct Block_byref *)_Block_allocator(src->size, false, isWeak);
        copy->flags = src->flags | _Byref_flag_initial_value; // non-GC one for caller, one for stack
        copy->forwarding = copy; // patch heap copy to point to itself (skip write-barrier)
        src->forwarding = copy;  // patch stack to point to heap copy
        copy->size = src->size;
        if (isWeak) {
            copy->isa = &_NSConcreteWeakBlockVariable;  // mark isa field so it gets weak scanning
        }
        if (src->flags & BLOCK_HAS_COPY_DISPOSE) {
            // Trust copy helper to copy everything of interest
            // If more than one field shows up in a byref block this is wrong XXX
            copy->byref_keep = src->byref_keep;
            copy->byref_destroy = src->byref_destroy;
            (*src->byref_keep)(copy, src);
        }
        else {
            // just bits.  Blast 'em using _Block_memmove in case they're __strong
            _Block_memmove(
                (void *)&copy->byref_keep,
                (void *)&src->byref_keep,
                src->size - sizeof(struct Block_byref_header));
        }
    }
    // already copied to heap
    else if ((src->forwarding->flags & BLOCK_NEEDS_FREE) == BLOCK_NEEDS_FREE) {
        latching_incr_int(&src->forwarding->flags);
    }
    // assign byref data block pointer into new Block
    _Block_assign(src->forwarding, (void **)destp);
}
```






