## 前言

iOS开发应该很熟悉weak关键字了,主要用于避免循环引用所带来的内存泄露问题.常见的使用场景有delegate, block等. 简单来说: 弱引用, 在对象释放后置位nil, 避免错误的内存访问.

本文主要是整理引用部分的文章,总结下加强自身理解.可以重点关注下引用链接的各位大神.

## runtime对_weak弱引用处理方式

```
NSObject *testObjc = [[NSObject alloc] init];
__weak NSObject *p = testObjc;
```
clang会对__weak进行转换
```
NSObject objc_initWeak(&p, 对象指针);
```
我们来看下objc_initWeak的具体实现
```
id objc_initWeak(id *location, id newObj) {
    // 查看对象实例是否有效
    // 无效对象直接导致指针释放
    if (!newObj) {
        *location = nil;
        return nil;
    }

    // 这里传递了三个 bool 数值
    // 使用 template 进行常量参数传递是为了优化性能
    return storeWeak<false/*old*/, true/*new*/, true/*crash*/>
        (location, (objc_object*)newObj);
}
```
实际上就是在storeWeak调用之前对指针指向的对象进行了一次有效性判断.
```
// HaveOld:     true - 变量有值
//             false - 需要被及时清理，当前值可能为 nil
// HaveNew:     true - 需要被分配的新值，当前值可能为 nil
//             false - 不需要分配新值
// CrashIfDeallocating: true - 说明 newObj 已经释放或者 newObj 不支持弱引用，该过程需要暂停
//             false - 用 nil 替代存储
template <bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id storeWeak(id *location, objc_object *newObj) {
    // 该过程用来更新弱引用指针的指向

    // 初始化 previouslyInitializedClass 指针
    Class previouslyInitializedClass = nil;
    id oldObj;

    // 声明两个 SideTable
    // ① 新旧散列创建
    SideTable *oldTable;
    SideTable *newTable;

    // 获得新值和旧值的锁存位置（用地址作为唯一标示）
    // 通过地址来建立索引标志，防止桶重复
    // 下面指向的操作会改变旧值
  retry:
    if (HaveOld) {
        // 更改指针，获得以 oldObj 为索引所存储的值地址
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (HaveNew) {
        // 更改新值指针，获得以 newObj 为索引所存储的值地址
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    // 加锁操作，防止多线程中竞争冲突
    SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);

    // 避免线程冲突重处理
    // location 应该与 oldObj 保持一致，如果不同，说明当前的 location 已经处理过 oldObj 可是又被其他线程所修改
    if (HaveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
        goto retry;
    }

    // 防止弱引用间死锁
    // 并且通过 +initialize 初始化构造器保证所有弱引用的 isa 非空指向
    if (HaveNew  &&  newObj) {
        // 获得新对象的 isa 指针
        Class cls = newObj->getIsa();

        // 判断 isa 非空且已经初始化
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) {
            // 解锁
            SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
            // 对其 isa 指针进行初始化
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // 如果该类已经完成执行 +initialize 方法是最理想情况
            // 如果该类 +initialize 在线程中 
            // 例如 +initialize 正在调用 storeWeak 方法
            // 需要手动对其增加保护策略，并设置 previouslyInitializedClass 指针进行标记
            previouslyInitializedClass = cls;

            // 重新尝试
            goto retry;
        }
    }

    // ② 清除旧值
    if (HaveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // ③ 分配新值
    if (HaveNew) {
        newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table, 
                                                      (id)newObj, location, 
                                                      CrashIfDeallocating);
        // 如果弱引用被释放 weak_register_no_lock 方法返回 nil 

        // 在引用计数表中设置若引用标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
            // 弱引用位初始化操作
            // 引用计数那张散列表的weak引用对象的引用计数中标识为weak引用
            newObj->setWeaklyReferenced_nolock();
        }

        // 之前不要设置 location 对象，这里需要更改指针指向
        *location = (id)newObj;
    }
    else {
        // 没有新值，则无需更改
    }

    SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

    return (id)newObj;
}
```
storeWeak的实现代码就复杂了一些,我们先分析代码涉及到的数据结构.

### SideTables

```
struct SideTable {
    // 保证原子操作的自旋锁
    spinlock_t slock;
    // 引用计数的 hash 表
    RefcountMap refcnts;
    // weak 引用全局 hash 表
    weak_table_t weak_table;
}
```
SideTable的结构大致如上,其内部的refcnts和weak_table稍后会分析.

```
newTable = &SideTables()[newObj];
```
最费解的可能是上面形式的代码了,转到定义可以看下具体实现.
```
static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```
简单理解,就是一个全局的Hash表,里面的内容都是SideTable的结构体,使用对象的内存地址作为key.其hash算法如下：
```
...
//如果是嵌入式系统StripeCount=8。我们这里StripeCount=64
enum { StripeCount = 64 };
...
static unsigned int indexForPointer(const void *p) {
    //这里是类型转换，不用在意
    uintptr_t addr = reinterpret_cast<uintptr_t>(p);

    //这里就是我们要找的Hash算法了
    return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}
```
可以看到最后的结果范围是在0-63之间,相当于SideTables一共创建了64个单元格来存储弱引用.哈希冲突？

### 分离锁
对象引用计数相关操作应该是原子性的,不然如果多线程同时读写一个对象的引用计数,那么就会造成数据错乱,失去了内存管理的意义.  
那每次操作的时候加锁吗？我们内存对象的数量是庞大的,需要非常频繁的操作SideTables,对整个hash表加锁的话,会造成性能问题,所以苹果这里采用了分离锁的技术.  
简单来说,假如数组里面有16个元素,分离锁的意思就是把这16个元素进行分组,比如按照每组4个元素就可分为4组.这样后续要读写元素的时候,就可以把加锁粒度控制到以小组为单位,而不是操作这个数组元素时,直接锁死整个大组.

### 从宿舍管理到SideTables

假设要给80个学生安排宿舍,同时要保证学生的财产安全应该怎么安排？(这个例子来源于引用的最后一篇，我觉得很好理解)

给80个学生分别安排80间宿舍,然后给每个宿舍大门上锁.  
这样太浪费资源了,而且会大致宿舍太多维护起来很费劲.

把80个学生分配到10间宿舍,每个宿舍住8个人.假设宿舍号分别是101,102,...,110.然后再给他们分配床位,01号床,02号床等.最后每个宿舍配一把锁来保护宿舍同学的财产安全.  
上面的例子就是现实生活中的例子,通过分离锁降低了锁的粒度.SideTables的管理机制就类似于宿舍管理.  

现在假如小明要找102宿舍2号床的人聊天.小明应该怎么做？
1. 找到宿舍楼(SideTables)的宿管,说自己要找10202(内存地址当做key)
2. 宿管带着小明去102宿舍即SideTables[10202]. 然后把102的门一锁lock,在他访问102期间不再允许其他访客访问102了.(这样只是阻塞了102的8个兄弟的访问,而不会影响整栋宿舍楼的访问)
3. 然后在宿舍里大喊一声:”2号床的兄弟在哪里？”table.refcnts.find(02)你就可以找到2号床的兄弟了
4.  等这个访客离开的时候会把房门的锁打开unlock，这样其他需要访问102的人就可以继续进来访问了。

```
SideTables == 宿舍楼
SideTable  == 宿舍
RefcountMap里存放着具体的床位
```
 苹果之所以需要创造SideTables的Hash冲突是为了把对象都放到宿舍里管理, 把锁的粒度缩小到一个宿舍SideTable.RefcountMap的工作是在找到宿舍以后帮助大家找到正确的床位的兄弟

### RefcountMap

具体类是DenseMap, 包含很多映射实例到其引用计数的键值对.支持Iterator快速查找遍历这些键值对.简单理解成一个Map其实就可以了,键为对象的内存地址,值为引用计数减一.

#### 获取引用计数
```
uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable *table = SideTable::tableForPointer(this);

    size_t refcnt_result = 1;
    
    spinlock_lock(&table->slock);
    RefcountMap::iterator it = table->refcnts.find(this);
    if (it != table->refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
    }
    spinlock_unlock(&table->slock);
    return refcnt_result;
}
```
这种获取引用计数的方式是针对引用计数过大,无法存储到isa指针的情况.  
从全局的SideTables获取到对应的SideTable.其中refcnts是存储引用计数的散列表.传入当前实例查找到对应的iterator,获取引用计数值,并在此基础上+1并将结果返回.这就是为什么之前说引用计数表存储的值为实际引用计数减一.

```
refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
```
这里为什么做了向右的位移操作？
```
#ifdef __LP64__
#   define WORD_BITS 64
#else
#   define WORD_BITS 32
#endif

// The order of these bits is important.
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  // MSB-ward of weak bit
#define SIDE_TABLE_RC_ONE            (1UL<<2)  // MSB-ward of deallocating bit
#define SIDE_TABLE_RC_PINNED         (1UL<<(WORD_BITS-1))

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)RefcountMap
```

第一个bit表示该对象是否有weak对象,如果没有,在析构释放内存时可以更快.  
第二个bit表示该对象是否正在析构.  
从第三个bit开始才是存储引用计数值的地方,所以这里要做向右位移两位的操作.而对引用计数的 +1 和 -1 可以使用 SIDE_TABLE_RC_ONE,还可以用 SIDE_TABLE_RC_PINNED 来判断是否引用计数值有可能溢出.  
_objc_rootRetainCount也不是绝对准确的, 对于已释放的对象以及不正确的对象地址，有时也返回 “1”.

#### 修改引用计数
```
bool 
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    SideTable *table = SideTable::tableForPointer(this);

    bool do_dealloc = false;

    if (spinlock_trylock(&table->slock)) {
        RefcountMap::iterator it = table->refcnts.find(this);
        if (it == table->refcnts.end()) {
            do_dealloc = true;
            table->refcnts[this] = SIDE_TABLE_DEALLOCATING;
        } else if (it->second < SIDE_TABLE_DEALLOCATING) {
            // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
            do_dealloc = true;
            it->second |= SIDE_TABLE_DEALLOCATING;
        } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
            it->second -= SIDE_TABLE_RC_ONE;
        }
        spinlock_unlock(&table->slock);
        if (do_dealloc  &&  performDealloc) {
            ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
        }
        return do_dealloc;
    }

    return sidetable_release_slow(table, performDealloc);
}
```

之前说过refcnts存放的是引用计数-1, 可以看这段代码理解下其意图.
```
it->second < SIDE_TABLE_DEALLOCATING

#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  // MSB-ward of weak bit
```

我们假设SIDE_TABLE_DEALLOCATING是00000010(一个字节的整数), 因为存放的是引用计数-1,所以最后一次调用release时. 此时的it->second的值就是00000000,进入代码块里面,因为其小于00000010这个标志位,所以就可以将do_dealloc置位true且将标志位置位析构状态. 然后去objc_msgSend对象的析构方法.

### weak_table_t
```
//全局弱引用表 键是不定类型对象的地址, 值是weak_entry_t 
struct weak_table_t {
    // 保存了所有指向指定对象的 weak 指针
    weak_entry_t *weak_entries;
    // 存储空间
    size_t    num_entries;
    // 参与判断引用计数辅助量
    uintptr_t mask;
    // hash key 最大偏移值
    uintptr_t max_hash_displacement;
};
```
weaktable实际上存储的是对象和弱引用的映射关系.对应的数据结构即weak_entry_t.num_entries负责记录table目前保存的entry数目,mask记录table的容量,max_hash_displacement记录table所有项的最大偏移量.  
针对hash碰撞,weaktable采用了开放式寻址来解决,所以entry实际的存储位置不一定是hash函数计算出来的位置.

我们下面从开放的API来理解weak_entries.

#### 容量控制
```
static void weak_grow_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Grow if at least 3/4 full.
	// Table 3/4及以上的空间已经被使用
    if (weak_table->num_entries >= old_size * 3 / 4) {
        weak_resize(weak_table, old_size ? old_size*2 : 64);
    }
}

static void weak_compact_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Shrink if larger than 1024 buckets and at most 1/16 full.
	//HashTable目前的大小不小于1024个weak_entry_t的空间，并且低于1/16的空间被占用。缩小后的空间是当前空间的1/8。
    if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
        weak_resize(weak_table, old_size / 8);
        // leaves new table no more than 1/2 full
    }
}
```
分别是扩充和缩小Hashtable的空间.大小都是2的N次方.
```
static void weak_resize(weak_table_t *weak_table, size_t new_size)
{
    size_t old_size = TABLE_SIZE(weak_table);

    weak_entry_t *old_entries = weak_table->weak_entries;
    weak_entry_t *new_entries = (weak_entry_t *)
        calloc(new_size, sizeof(weak_entry_t));

    weak_table->mask = new_size - 1;
    weak_table->weak_entries = new_entries;
    weak_table->max_hash_displacement = 0;
    weak_table->num_entries = 0;  // restored by weak_entry_insert below
    
    if (old_entries) {
        weak_entry_t *entry;
        weak_entry_t *end = old_entries + old_size;
        for (entry = old_entries; entry < end; entry++) {
            if (entry->referent) {
                weak_entry_insert(weak_table, entry);
            }
        }
        free(old_entries);
    }
}
```
代码比较简单,即重新分配内存空间.重置结构体其余属性.然后把老的entries的值 插入到新的entries.

####  增删查
```
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
    weak_entry_t *weak_entries = weak_table->weak_entries;
    assert(weak_entries != nil);

    size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_entries[index].referent != nil) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_entries);
        hash_displacement++;
    }

    weak_entries[index] = *new_entry;
    weak_table->num_entries++;

    if (hash_displacement > weak_table->max_hash_displacement) {
        weak_table->max_hash_displacement = hash_displacement;
    }
}
```

插入一个entry(对象和弱引用的映射关系)到weak_table中.  
`size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);`
hash_pointer是对指针的地址进行一系列的移位,异或等运算并返回.begin是hash_pointer的返回值和mask进行与运算的结果.这里为什么要与一次呢？ 
weak_table的size始终是2的N次方,而mask的值是size-1.所以mask的二进制后N位都是1,之前的位数都是0. 比如size是32,那么mask就是00011111.hashpointer的返回值和mask进行与运算以后,最后结果肯定在[0,32]这个区间内,即刚好在weak_table的合法范围内.  
如果发生了hash冲突怎么办呢？
```
 while (weak_entries[index].referent != nil) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_entries);
        hash_displacement++;
    }
```
hash冲突即获取到的referent不为nil.那么就index++,同时得保证不越界,如果找了一圈没找到就bad_weak_table.否则找到空位以后记录下偏移量.执行插入,与目前最大偏移量比较.如果更大就改变最大偏移量的值.

```
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    assert(referent);

    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t begin = hash_pointer(referent) & weak_table->mask;
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
}
```
查找的流程和插入类似,仔细跟下函数实现即可.

```
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{
    // remove entry
    if (entry->out_of_line()) free(entry->referrers);
    bzero(entry, sizeof(*entry));
    weak_table->num_entries--;
    weak_compact_maybe(weak_table);
}
```
删除的方法较为简单,内存清除,num_entries的计数减1,视情况缩小weak_table空间.

#### runtime接口
```
id weak_register_no_lock(weak_table_t *weak_table, id referent, 
                         id *referrer, bool crashIfDeallocating);

void weak_unregister_no_lock(weak_table_t *weak_table, id referent, id *referrer);

bool weak_is_registered_no_lock(weak_table_t *weak_table, id referent);

void weak_clear_no_lock(weak_table_t *weak_table, id referent);
```
下面分析下具体实现.
```
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
```
Tagged Pointer等部分略过,直接看末尾注册部分代码.首先通过之前分析过的方法来查找referent(对象和弱引用的映射关系是否存在).如果存在说明又有新的弱引用指针指向了对象, 直接执行append_referrer操作.如果不存在则新增映射关系.

```
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;

    if (!referent) return;

    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }

        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
}
```

该函数的整体逻辑也相对简单,首先找到object对象的entry,并且删除对应的weak引用,而后判断entry中的weak引用是否已经为空.如果是则从weak_table中将其删除.

```
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    weak_entry_remove(weak_table, entry);
}
```
删除一个object所有的weak引用,并且清理其空间.最后从weak_table中将其对应的entry删除.


###  weak_entry_t
```
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```

之前我们分析runtime接口的时候，只是根据函数名等简单分析了下作用，现在看了weak_entry_t就可以连通理解下.   
首先SideTable是利用分离锁的概念分出的一些对象的集合.  
主要成员有refcntMap,这个和我们之前理解的键值对Map一样,键是对象,值是引用计数.   
然后是weakTable, weakTable是一个哈希表,简单来说就是数组+特殊定位index的方法.   weakTable数组的元素是weakentry, 这个结构体的成员变量如上面代码所示.
weakentry首先有一个对象成员变量referent.  
```
NSObject *testObjc = [[NSObject alloc] init];
__weak NSObject *p = testObjc;
```
即上面的testobjc,然后是一个联合体,针对不同数量的weak,来选择不同的实现.
可以直接是一个内联数组，也可以是hash表.hash表存储的是weak对象即上面代码的p.    
如果对象的弱引用数目较少(<=WEAK_INLINE_COUNT 即4),弱引用会因此保存到一个inline数组里.这个inline数组会在entry初始化的时候分配好, 而不是需要的时候再申请新的内存.超过了数目的话,则使用hashtable的方式来实现.

#### append_referrer

```
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    if (! entry->out_of_line()) {
        // Try to insert inline.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }

        // Couldn't insert inline. Allocate out of line.
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        // This constructed table is invalid, but grow_refs_and_insert
        // will fix it and rehash it.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            new_referrers[i] = entry->inline_referrers[i];
        }
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
    }

    assert(entry->out_of_line());

    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        return grow_refs_and_insert(entry, new_referrer);
    }
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != nil) {
        hash_displacement++;
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
    }
    if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
    }
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
```

1. 弱引用数目较少, 直接使用inline操作.把对象加入数组  
2. 使用outline. 如果outline数组的使用率在75%及以上，那么调用grow_refs_and_insert函数进行扩充，并且插入新的弱引用。否则就直接进行hash运算插入，过程和weak_table_t的插入过程基本相同  

```
__attribute__((noinline, used))
static void grow_refs_and_insert(weak_entry_t *entry, 
                                 objc_object **new_referrer)
{
    assert(entry->out_of_line());

    size_t old_size = TABLE_SIZE(entry);
    size_t new_size = old_size ? old_size * 2 : 8;

    size_t num_refs = entry->num_refs;
    weak_referrer_t *old_refs = entry->referrers;
    entry->mask = new_size - 1;
    
    entry->referrers = (weak_referrer_t *)
        calloc(TABLE_SIZE(entry), sizeof(weak_referrer_t));
    entry->num_refs = 0;
    entry->max_hash_displacement = 0;
    
    for (size_t i = 0; i < old_size && num_refs > 0; i++) {
        if (old_refs[i] != nil) {
            append_referrer(entry, old_refs[i]);
            num_refs--;
        }
    }
    // Insert
    append_referrer(entry, new_referrer);
    if (old_refs) free(old_refs);
}
```

函数首先对outline数组进行扩充，容量是原来的两倍。而后依次将老数组中的元素hash插入到新数组中，最终hash插入新的引用

#### remove_referrer
```
static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
    if (! entry->out_of_line()) {
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == old_referrer) {
                entry->inline_referrers[i] = nil;
                return;
            }
        }
        _objc_inform("Attempted to unregister unknown __weak variable "
                     "at %p. This is probably incorrect use of "
                     "objc_storeWeak() and objc_loadWeak(). "
                     "Break on objc_weak_error to debug.\n", 
                     old_referrer);
        objc_weak_error();
        return;
    }

    size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != old_referrer) {
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
        hash_displacement++;
        if (hash_displacement > entry->max_hash_displacement) {
            _objc_inform("Attempted to unregister unknown __weak variable "
                         "at %p. This is probably incorrect use of "
                         "objc_storeWeak() and objc_loadWeak(). "
                         "Break on objc_weak_error to debug.\n", 
                         old_referrer);
            objc_weak_error();
            return;
        }
    }
    entry->referrers[index] = nil;
    entry->num_refs--;
}

```

remove_referrer函数负责删除一个弱引用。函数首先处理inline数组的情况，直接将对应的弱引用项置空。如果使用了outline数组，则通过hash找到要删除的项，并直接删除。过程和weak_table_t对应的操作基本相同  

### 释放

释放时,调用clearDeallocating函数
```
void 
objc_object::sidetable_clearDeallocating()
{
    SideTable& table = SideTables()[this];

    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
            weak_clear_no_lock(&table.weak_table, (id)this);
        }
        table.refcnts.erase(it);
    }
    table.unlock();
}
```
对象析构的时候，会先找到对应的SideTable，然后在RefcountMap查找到其引用计数部分,看起标志位是否支持SIDE_TABLE_WEAKLY_REFERENCED.支持的话就去调用weak_clear_no_lock.weak_clear_no_lock上面已经有具体分析了.删除一个object所有的weak引用,并且清理其空间.把所有weak对象置位nil.最后从weak_table中将其对应的entry删除，



## 总结

Runtime维护一个全局的SideTables表. 利用Hash算法+分离锁将对象进行分组. SideTable维护引用计数表和弱引用对象依赖表. 初始化和添加若引用时,调用storeWeak创建或者修改 对象和弱引用的 映射关系. 释放时,调用deallocating函数,先获取到weak指针的集合,再遍历把其中的数据设置为nil.最后从weaktable中移除掉当前的entry.  



## 引用

[iOS weak关键字漫谈](https://zhuanlan.zhihu.com/p/27832890)  
[weak 弱引用的实现方式](http://www.desgard.com/iOS-Source-Probe/Objective-C/Runtime/weak%20%E5%BC%B1%E5%BC%95%E7%94%A8%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F.html)  
[Objective-C 引用计数原理](http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/)  
[OC Runtime之Weak](https://www.jianshu.com/p/045294e1f062)   
[iOS管理对象内存的数据结构以及操作算法](https://www.jianshu.com/p/ef6d9bf8fe59)  
