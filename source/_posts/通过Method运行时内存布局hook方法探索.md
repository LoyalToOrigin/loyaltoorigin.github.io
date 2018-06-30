---
title: 通过Method运行时内存布局hook方法探索
date: 2018-06-30
categories: 
---

在iOS开发中, Method Swizzling想必大家都不陌生, 可以以此来对方法进行hook, 做一些我们希望做的事情, 比如页面进入退出, 可以对viewWillAppear及viewWillDisappear进行hook, 从而进行一些埋点日志相关的事情。

那么, Method Swizzling的原理到底是怎样的呢? 这个问题, 即使没自己研究过, 大多数人也有所耳闻, 简单来说, 无非就是修改方法的imp指向, 让其指向我们hook的方法。如果是这样的话, 我们是否可以不用Runtime提供的API如method_setImplementation、method_exchangeImplementation等函数而通过对象及方法的内存布局来实现呢? 答案是肯定的, 下面便是我在此过程中的一些探索和理解。

本文描述大部分内容对开发没有太大帮助, 但是对于更加了解运行时方法调用有一定帮助。

# 直接赋值Method的IMP进行hook

要想通过方法的内存布局来修改, 一定要对方法的内存布局有所了解, 查看源码可以知道Method的内存布局如下所示:

```
struct method_t {
    SEL name;
    const char *types;
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

上面结构中, 很容易就找到我们想要的东西IMP, 话不多少, 赶紧进行hook。

```
@implementation Person

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class aClass = [self class];
        
        SEL originalSelector = @selector(sayHello);
        Method originalMethod = class_getInstanceMethod(aClass, originalSelector);
       
        struct method_t *method = (struct method_t *)originalMethod;
        method->imp = (IMP)hookedSayHello;
    });
}

- (void)sayHello {
    NSLog(@"Hello, everybody!");
}

void hookedSayHello (id self, SEL _cmd, ...) {
    NSLog(@"This is hooked sayHello");
}

@end
```

然后再main.m中调用:

```
    Person *person = [[Person alloc] init];
    [person sayHello];
```

### 遇到的问题, 还是调用原来的方法实现

此时却发现, 打印出来的却和我想象不太一样, 仍然是调用了原来的sayHello方法, 而且打个断点发现method的imp指针也确实指向了 void hookedSayHello (id self, SEL _cmd, ...) 这个函数,  这确实有些让人捉摸不透。 

![直接修改方法imp](http://p28r7eh75.bkt.clouddn.com/%E9%80%9A%E8%BF%87Method%E8%BF%90%E8%A1%8C%E6%97%B6%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80hook%E6%96%B9%E6%B3%95%E6%8E%A2%E7%B4%A2/hooked_imp.png)

### 浅尝辄止--method _setImplementation

于是怀疑人生的我, 又使用Runtime提供的API method_setImplementation进行相同操作, 发现和以往一样, 毫无问题, 那么一定是做了一些处理, 查其源码, 发现了一个很可疑的函数 flushCaches, 见名知意, 清除缓存。

```
static IMP 
_method_setImplementation(Class cls, method_t *m, IMP imp)
{
    runtimeLock.assertWriting();

    if (!m) return nil;
    if (!imp) return nil;

    IMP old = m->imp;
    m->imp = imp;

    // Cache updates are slow if cls is nil (i.e. unknown)
    // RR/AWZ updates are slow if cls is nil (i.e. unknown)
    // fixme build list of classes whose Methods are known externally?

    flushCaches(cls); 

    updateCustomRR_AWZ(cls, m);

    return old;
}


/***********************************************************************
* _objc_flush_caches
* Flushes all caches.
* (Historical behavior: flush caches for cls, its metaclass, 
* and subclasses thereof. Nil flushes all classes.)
* Locking: acquires runtimeLock
**********************************************************************/
static void flushCaches(Class cls)
{
    runtimeLock.assertWriting();

    mutex_locker_t lock(cacheUpdateLock);

    if (cls) {
        foreach_realized_class_and_subclass(cls, ^(Class c){ // 遍历子类
            cache_erase_nolock(c);
        });
    }
    else {
        foreach_realized_class_and_metaclass(^(Class c){
            cache_erase_nolock(c);
        });
    }
}

// Reset this entire cache to the uncached lookup by reallocating it.
// This must not shrink the cache - that breaks the lock-free scheme.
void cache_erase_nolock(Class cls)
{
    cacheUpdateLock.assertLocked();

    cache_t *cache = getCache(cls);

    mask_t capacity = cache->capacity();
    if (capacity > 0  &&  cache->occupied() > 0) {
        auto oldBuckets = cache->buckets();
        auto buckets = emptyBucketsForCapacity(capacity);
        cache->setBucketsAndMask(buckets, capacity - 1); // also clears occupied

        cache_collect_free(oldBuckets, capacity);
        cache_collect(false);
    }
}

```

如上述源码可知, 在flushCaches函数中, 这个函数会把当前类本身, 当前类的元类以及当前类的子类的方法缓存全部清空, 这里我们也可以自己验证一下, 

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class aClass = [self class];
        
        SEL originalSelector = @selector(sayHello);
              
        Method originalMethod = class_getInstanceMethod(aClass, originalSelector);
        
        
//        method_setImplementation(originalMethod, (IMP)hookedSayHello); //Runtime API, 可以发现cache被清除了, 可以打开注释, 验证结果
        
        struct method_t *method = (struct method_t *)originalMethod;
//        method->imp = (IMP)hookedSayHello; // 直接复制imp指针

        struct my_objc_class *clz = (__bridge struct my_objc_class *)aClass;
        uint32_t cacheCount = clz->cache.capacity();
        NSLog(@"cacheCount : %d", cacheCount);
        
        
        for (NSInteger i = 0; i < cacheCount; i++) {
            char *key = (char *)((clz->cache._buckets + i)->_key);
            // 这里设置一下
            printf("%ld - %s\n", i, key); // 测试
    });
}

```

当调用Runtime API method_setImplementation, 打印如下图所示:
![调用Runtime API method_setImplementation,cache被清除](http://p28r7eh75.bkt.clouddn.com/%E9%80%9A%E8%BF%87Method%E8%BF%90%E8%A1%8C%E6%97%B6%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80hook%E6%96%B9%E6%B3%95%E6%8E%A2%E7%B4%A2/method_setImplementation_cache.png)
当直接给imp指针赋值, 打印如下图所示:
![imp指针赋值,cache没被清除](http://p28r7eh75.bkt.clouddn.com/%E9%80%9A%E8%BF%87Method%E8%BF%90%E8%A1%8C%E6%97%B6%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80hook%E6%96%B9%E6%B3%95%E6%8E%A2%E7%B4%A2/direct_imp_cache.png)

可以看出, 当直接给imp指针复制, 不清除方法缓存, 其中打印的sayHello正是我们hook的方法, 之前的疑惑也一扫而空, 虽然方法的imp指向发生了改变, 但是方法缓存中的sayHello对应的imp并没有发生改变。

我们知道, Objective-C通过方法缓存来提升方法调用速度, 缓存中找不到, 再去类对象的方法列表中去查找, 调用后便加入到方法缓存中, 这点也可以通过objc_msgSend的源码来确认, objc_msgSend的源码是汇编实现的, 即使看不懂汇编也没事, 通过旁边的注释, 大概来看出来调用流程: 在方法缓存中寻找, 找到直接返回方法IMP, 否则调用__objc_msgSend_uncached, 去方法列表中查找。


```
/// objc_msgSend, 除去一些nil验证检测后, 调用 CacheLookup LOOKUP
LLookup_GetIsaDone:
	CacheLookup LOOKUP		// returns imp


/// CacheLookup
	.macro CacheHit
.if $0 == NORMAL
	MESSENGER_END_FAST
	br	x17			// call imp
.elseif $0 == GETIMP
	mov	x0, x17			// return imp
	ret
.elseif $0 == LOOKUP
	ret				// return imp via x17
.else
.abort oops
.endif
.endmacro

.macro CheckMiss
	// miss if bucket->sel == 0
.if $0 == GETIMP
	cbz	x9, LGetImpMiss
.elseif $0 == NORMAL
	cbz	x9, __objc_msgSend_uncached
.elseif $0 == LOOKUP
	cbz	x9, __objc_msgLookup_uncached

```

# 作怪到底--自己修改方法缓存对应的imp

既然都到这里, 不妨尝试自己去修改方法缓存中对应imp。其实从Objective-C Runtime层面来说, 对象、方法、block等都是以结构体的形式存在内存中, 想去改对象的属性, 方法的实现会是block的实现, 都是要对它们的内存布局有所了解。

前面的分析把疑惑基本解决了, 现在要做的就比较简单是了, 只需要将方法缓存以及其他需要用到的结构体如对象、方法等的结构抽出来, 自己声明一个结构体, 把需要用上的成员变量和方法带上即可, 不需要用上可以直接删除。

```
struct bucket_t {
    cache_key_t _key;
    IMP _imp;
};

struct cache_t {
    bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
public:
    struct bucket_t *buckets();
    mask_t mask();
    mask_t occupied();
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void initializeToEmpty();
    
    mask_t capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();
    
    static size_t bytesForCapacity(uint32_t cap);
    static struct bucket_t * endMarker(struct bucket_t *b, uint32_t cap);
    
    void expand();
    void reallocate(mask_t oldCapacity, mask_t newCapacity);
    struct bucket_t * find(cache_key_t key, id receiver);
    
    static void bad_cache(id receiver, SEL sel, Class isa) __attribute__((noreturn));
};

```

接下来, 只需要将load方法中添加一点代码进行验证即可:

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class aClass = [self class];
//        Class aClass = self; // 不给self发消息, cache不会生成, 结果就和我们的预想一样
        
        SEL originalSelector = @selector(sayHello);
        
        Method originalMethod = class_getInstanceMethod(aClass, originalSelector);
        
//        method_setImplementation(originalMethod, (IMP)hookedSayHello); //Runtime API, 可以发现cache被清除了, 可以打开注释, 验证结果
        
        struct method_t *method = (struct method_t *)originalMethod;
        method->imp = (IMP)hookedSayHello;
        
        // cache问题, 因为 已经和 imp缓存了, 直接会调用原来方法
        // method_setImplementation 中有个函数 flushCache -> cache_erase_nolock, 会重新设置 cache
        
        // 修改cache
        struct my_objc_class *clz = (__bridge struct my_objc_class *)aClass;
        uint32_t cacheCount = clz->cache.capacity();
        NSLog(@"cacheCount : %d", cacheCount);
        
        
        for (NSInteger i = 0; i < cacheCount; i++) {
            char *key = (char *)((clz->cache._buckets + i)->_key);
            // 这里设置一下
            printf("%ld - %s\n", i, key); // 测试
            
            if (key) {
                NSString *selectorName = [NSString stringWithUTF8String:key];

                if ([selectorName isEqualToString:@"sayHello"]) {
                    (clz->cache._buckets + i)->_imp = (IMP)hookedSayHello;
                }
            }
        }
    });
}

```
![自己修改cache](http://p28r7eh75.bkt.clouddn.com/%E9%80%9A%E8%BF%87Method%E8%BF%90%E8%A1%8C%E6%97%B6%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80hook%E6%96%B9%E6%B3%95%E6%8E%A2%E7%B4%A2/direct_imp_cache_and_change_cache_imp.png)

发现打印的确实是我们希望的实现, 当然这里只是一个简单的类, 对于有子类的情况没做验证, 如果有子类的情况下, 还是比较复杂的, 对于子类是否实现了该方法也是有区别的, 这也许也是 method_setImplementation 直接暴力地将当前类和子类的缓存都清空的原因吧!


# 总结

通过本次探索, 对方法调用以及底层的一些流程有了一定的了解, 虽然对于开发确实没太大帮助, 但对于理解底层机制有一定帮助。在日常学习中, 可以配合源码, 通过自己的尝试, 一定可以对相关知识有更深刻地理解。

代码地址: [https://github.com/LoyalToOrigin/HookMethodWithLayout]()





