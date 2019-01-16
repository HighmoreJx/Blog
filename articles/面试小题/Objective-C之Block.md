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
int main(int argc, const char * argv[]) {
    int i = 5;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_07_p4zsgcc12dsbg_1gbhy9km5m0000gn_T_main_860511_mi_0, ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, i)));
    return 0;
}
```

#### NSConcreteMallocBlock
