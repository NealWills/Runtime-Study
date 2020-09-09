
---

### `[[Objc alloc] init]`

一般情况下，我们初始化一个对象时会调用 `[[Obj alloc] init]` 来进行初始化，就算是对其进行重写为 `[[Objc alloc] initWithXXX:xxx]` 还是会在内部去调用 `[[super alloc] init]` 方法，那么，alloc和init到底扮演了一个什么样对角色呢？

---

#### `alloc`

打开xcode的汇编模式，然后在初始化`NSObject对象`时打一个断点，运行后，xcode会停在我们打得断点的地方，并展示出当前汇编进程：
![alloc汇编进程](https://upload-images.jianshu.io/upload_images/11398671-9443f8948ac7ef3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，在12行调用来一个方法`objc_alloc_init`,我们去到`runtime`的源码(781版本)中搜索这个方法：
```
// Calls [[cls alloc] init].
id
objc_alloc_init(Class cls)
{
    return [callAlloc(cls, true/*checkNil*/, false/*allocWithZone*/) init];
}
```
然后跟着一路点进去：
```
// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
#if __OBJC2__
    if (slowpath(checkNil && !cls)) return nil;
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif

    // No shortcuts available.
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));
}
```
因为现在都是使用的objc2版本也就是`Modern`,所以会走`_objc_rootAllocWithZone`这个方法，所以我们跟着点进去：
```
id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone __unused)
{
    // allocWithZone under __OBJC2__ ignores the zone parameter
    return _class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC);
}
```


```

/***********************************************************************
* class_createInstance
* fixme
* Locking: none
*
* Note: this function has been carefully written so that the fastpath
* takes no branch.
**********************************************************************/
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    // 如果指定了内存空间，走这个
    if (zone) {
        // malloc表示在記憶體的動態儲存區中分配一塊長度為“size”位元組的連續區域，返回該區域的首地址；
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        // calloc表示在記憶體的動態儲存區中分配n塊長度為“size”位元組的連續區域，返回首地址。
        obj = (id)calloc(1, size);
    }
    // slowpath判断是否为假
    // fastpath判断是否为真
    
    // 这里表示!obj是否为假
    if (slowpath(!obj)) {
        // 如果!obj为假，则经过处理后返回这块内存地址isa
        // 其实就是obj不为空
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    // 如果没有指定内存空间，走这个（对应上面的 !zone）
    if (!zone && fast) {
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    // 这里表示!hasCxxCtor为真
    if (fastpath(!hasCxxCtor)) {
        // 如果!hasCxxCtor为真，返回obj，也就是首地址
        // 其实就是 hasCxxCtor 为假
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

```
/***********************************************************************
* object_cxxConstructFromClass.
* Recursively call C++ constructors on obj, starting with base class's 
*   ctor method (if any) followed by subclasses' ctors (if any), stopping 
*   at cls's ctor (if any).
* Does not check cls->hasCxxCtor(). The caller should preflight that.
* Returns self if construction succeeded.
* Returns nil if some constructor threw an exception. The exception is 
*   caught and discarded. Any partial construction is destructed.
* Uses methodListLock and cacheUpdateLock. The caller must hold neither.
*
* .cxx_construct returns id. This really means:
* return self: construction succeeded
* return nil:  construction failed because a C++ constructor threw an exception
**********************************************************************/
id 
object_cxxConstructFromClass(id obj, Class cls, int flags)
{
    ASSERT(cls->hasCxxCtor());  // required for performance, not correctness

    id (*ctor)(id);
    Class supercls;

    supercls = cls->superclass;

    // Call superclasses' ctors first, if any.
    if (supercls  &&  supercls->hasCxxCtor()) {
        bool ok = object_cxxConstructFromClass(obj, supercls, flags);
        if (slowpath(!ok)) return nil;  // some superclass's ctor failed - give up
    }

    // Find this class's ctor, if any.
    ctor = (id(*)(id))lookupMethodInClassAndLoadCache(cls, SEL_cxx_construct);
    if (ctor == (id(*)(id))_objc_msgForward_impcache) return obj;  // no ctor - ok
    
    // Call this class's ctor.
    if (PrintCxxCtors) {
        _objc_inform("CXX: calling C++ constructors for class %s", 
                     cls->nameForLogging());
    }
    if (fastpath((*ctor)(obj))) return obj;  // ctor called and succeeded - ok

    supercls = cls->superclass; // this reload avoids a spill on the stack

    // This class's ctor was called and failed.
    // Call superclasses's dtors to clean up.
    if (supercls) object_cxxDestructFromClass(obj, supercls);
    if (flags & OBJECT_CONSTRUCT_FREE_ONFAILURE) free(obj);
    if (flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
        return _objc_callBadAllocHandler(cls);
    }
    return nil;
}
```
---

分析源码，我们可以画出一条流程图
![alloc流程](https://upload-images.jianshu.io/upload_images/11398671-c4fdc0952a4f6db6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后，我们得到了一块内存的首地址，这就是alloc的作用

#### `init`

同上，通过搜索源码，我们可以找到init的源码如下：
```

// Replaced by CF (throws an NSException)
+ (id)init {
    return (id)self;
}

- (id)init {
    return _objc_rootInit(self);
}
```
实际上，init只是创建了一个指针，这个指针指向了alloc得到的那块内存，这就是init的作用