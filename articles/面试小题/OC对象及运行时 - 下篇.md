

本篇主要讲下应用及延展.  

## rumtime提供的操作函数

[苹果官方API](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)可以查阅到所有相关操作函数.  
[Objective-C Runtime 运行时之一：类与对象](http://southpeak.github.io/2014/10/25/objective-c-runtime-1/)简要介绍了一些函数的用途, 可以结合看下.   

## 关联对象(Associated Object)与分类

```
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```





中篇的时候说过, ivar是编译期的产物(运行时从无到有产生一个类例外), 不能等到运行时再往里面加ivar.  
那category岂不是就不能添加实例变量了?  

关联对象就很适用于该场景提到的问题.  





## 引用

[Objective-C runtime - 分类与关联对象](http://vanney9.com/2017/06/07/objective-c-runtime-category/)
