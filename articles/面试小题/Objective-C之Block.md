
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



