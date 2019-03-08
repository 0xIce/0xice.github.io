## 1. 自动引用计数

### 1.1 什么是自动引用计数

内存管理中对引用采取自动计数的技术。让编译器来进行内存管理，无需手动键入`retain`或者`release`代码，很大程度上减少了开发程序的工作量。

### 1.2 内存管理/引用计数

#### 1.2.1 内存管理的思考方式

- 自己生成的对象，自己所持有 (alloc/new/copy/mutableCopy)
- 非自己生成的对象，自己也能持有 (retain)
- 不再需要自己持有的对象时释放 (release)
- 非自己持有的对象无法释放
- 没有被持有的对象废弃 (delloc)

`这些内存管理的方法，实际上不包括在该语言中，而是包含在Cocoa框架中用于OSX、iOS应用开发`

#### 1.2.2  alloc/retain/release/dealloc 实现

##### 1.2.2.1 GNUstep实现

代码在[GNUstep](https://github.com/gnustep/libs-base)

**alloc**

```objective-c
+ (id) alloc
{
  return [self allocWithZone: NSDefaultMallocZone()];
}

+ (id) allocWithZone: (NSZone*)z
{
  return NSAllocateObject(self, 0, z);
}

// 简化版的NSAllocateObject
inline id
NSAllocateObject(Class aClass, NSUInteger extraBytes, NSZone *zone)
{
  id	new;
  int	size;
  size = class_getInstanceSize(aClass) + extraBytes + sizeof(struct obj_layout);
  new = NSZoneMalloc(zone, size);
  if (new != nil)
    {
      memset (new, 0, size);
      new = (id)&((obj)new)[1];
      object_setClass(new, aClass);
      AADD(aClass, new);
    }
  return new;
}

struct obj_layout {
  gsrefcount_t	retained;
};
typedef	struct obj_layout *obj;
```

`NSAllocateObject`通过`NSZoneMalloc`分配了对象所需的空间，初始化为0，`obj_layout`中的retained保存引用对象的引用计数值， 设置了对应的`class`,返回对象。

**retainCount**

```objective-c
- (NSUInteger) retainCount
{
  return getRetainCount(self);
}

size_t getRetainCount(id anObject)
{
   return object_getRetainCount_np_internal(anObject);
}

size_t object_getRetainCount_np_internal(id anObject)
{
  return ((obj)anObject)[-1].retained + 1;
}

/**
 * The class implementation of the retainCount method always returns
 * the maximum unsigned integer value, as classes can not be deallocated
 * the retain count mechanism is a dummy system for them.
 */
+ (NSUInteger) retainCount
{
  return UINT_MAX;
}
```

alloc之后的内存全部置0，获取的retained为0，加一之后返回，alloc之后的对象的rc为1

**retain**

```objective-c
- (id) retain
{
	return retain_fast(self)
}

inline void
retain_fast(id anObject)
{
	((obj)anObject)[-1].retained++;
	return anObject;
}
```

retain方法就是让obj的retained++

**release**

```objective-c
- (oneway void) release
{
  release_fast(self);
}

static void release_fast(id anObject)
{
    objc_release_fast_np_internal(anObject);
}

static void objc_release_fast_np_internal(id anObject)
{
    if (release_fast_no_destroy(anObject))
    {
        [anObject dealloc];
    }
}

static BOOL release_fast_no_destroy(id anObject)
{
	return objc_release_fast_no_destroy_internal(anObject);
}

static BOOL objc_release_fast_no_destroy_internal(id anObject)
{
    pthread_mutex_t *theLock = GSAllocationLockForObject(anObject);

    pthread_mutex_lock(theLock);
    if (((obj)anObject)[-1].retained == 0)
      {
#  ifdef OBJC_CAP_ARC
        objc_delete_weak_refs(anObject);
#  endif
        pthread_mutex_unlock(theLock);
        return YES;
      }
    else
      {
        ((obj)anObject)[-1].retained--;
        pthread_mutex_unlock(theLock);
        return NO;
      }
	return NO;
}
```

最终会判断retained值，大于0时减1，等于0时调用dealloc方法。

**dealloc**

```objective-c
- (void) dealloc
{
  NSDeallocateObject(self);
}

inline void
NSDeallocateObject(id anObject)
{
  Class aClass = object_getClass(anObject);

  if ((anObject != nil) && !class_isMetaClass(aClass))
    {
      obj	o = &((obj)anObject)[-1];
      NSZone	*z = NSZoneFromPointer(o);
	  object_dispose(anObject);
	  object_setClass((id)anObject, (Class)(void*)0xdeadface);
	  NSZoneFree(z, o);
    }
  return;
}
```

释放alloc分配的内存块。

**总结**

- 在oc对象中存有引用计数这一整数值
- 跳用alloc或者retain方法后，引用计数值加1
- 调用release方法后，引用计数值减1
- 引用计数值为0时，调用dealloc方法废弃对象

##### 1.2.2.2  苹果的实现

**alloc**

```objective-c
+ (id)alloc {
    return _objc_rootAlloc(self);
}

id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
```



`+alloc -> allocWithZone: -> class_createInstance -> calloc`

**retainCount/retain/releasse**

苹果的引用计数值存储在一个散列表中，对应的操作是通过`obj`获取引用计数表，操作对应的引用计数值。

##### 1.2.2.3 对比

**通过内存块头部管理引用计数的好处**

- 少量代码即可完成
- 能够统一管理引用计数内存块和对象用内存块

**通过引用计数表管理引用计数的好处**

- 对象用内存块的分配无需考虑内存块头部
- 引用计数表记录中存有内存块地址，可从各个记录中追溯到各个对象的内存块

#### 1.2.3 autorelease实现

##### 1.2.3.1 GNUstep实现







##### 1.2.3.2 苹果实现

























end









