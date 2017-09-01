---
title: "再谈Objective-C弱引用容器的实现-关联对象实现weak"
date: 2017-09-02T03:36:00+08:00
draft: false
---

之前写了一篇[文章](https://nemocdz.github.io/My-blog/post/objective-c%E5%BC%B1%E5%BC%95%E7%94%A8%E5%AE%B9%E5%99%A8%E5%AE%9E%E7%8E%B0%E6%96%B9%E6%A1%88%E6%80%BB%E7%BB%93/)总结了OC中弱引用容器实现，在小米面试中提到其中CFFoundation的做法，面试官问了我一个问题，这样实现后在这些元素在被销毁后，还保留在容器中会有什么问题么？我马上意识到，这些元素会变成野指针，且之前只实现了引用计数的不变，而没有实现Weak特质，也就是没有在销毁后置nil，也没有被移除，那么容器外界再访问时就会崩溃。看来之前考虑得还是太片面，也没有做更周全的实验。

所以看了Runtime源码和文章后，订正弱引用容器的一些实现方法。

### Runtime源码的weak关键字实现

源码基于[Runtime-709](https://opensource.apple.com/tarballs/objc4/)分析，当我们使用``__weak``关键字时，实际上调用的是NSObject.mm中的``objc_initWeak()``方法，然后进入核心方法``storeWeak``，传入核心参数是location（弱引用指针的地址）和newobj（弱引用指针应该指向的）核心方法源码如下：

```c++
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
//template关键字类似于泛型，传入三个参数其实都是bool类型，为各种情况提供组合优化
static id //返回id类型
storeWeak(id *location, objc_object *newObj)
{   
    
    assert(haveOld  ||  haveNew);
    //新值没有的情况应该被上层的方法拦截，所以这里有个断言
    if (!haveNew) assert(newObj == nil);
    
    
    Class previouslyInitializedClass = nil;
    id oldObj;
    //SideTable结构是引用计数表，记录着对象的weak表，目的就是为了获取weak表
    SideTable *oldTable;
    SideTable *newTable;

 retry:
    if (haveOld) {
        oldObj = *location;//因为是个指向地址的指针，用*解引用返回地址的对象
        oldTable = &SideTables()[oldObj];//获得旧值对象所在的表的指针
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];//获得新值对象生意所在的表的指针
    } else {
        newTable = nil;
    }
    
    //加锁
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
    //再次确认保证线程安全
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // 防止弱引用机制的死锁并用init构造保证弱引用的isa指针非空
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));
            previouslyInitializedClass = cls;
            goto retry;
        }
    }

    //清除旧址
    if (haveOld) {
        //在weakTable中反注册
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    //分配新值
    if (haveNew) {
        //在weakTable中注册新值并返回对象
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);


        // 对TaggedPointer的优化
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        *location = (id)newObj;
    }
    else {
    }
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}

```

而WeakTable是一个Hash表设计，以对象的地址为key，value是所有指向这个对象的weak指针的地址集合。通过这种设计，在废弃对象时，可以通过weak表快速找到value即所有weak指针并统一设置为nil并删除该记录。

**所以苹果对于weak的实现其实类似于通知的实现，指明谁（weak指针）要监听谁（赋值对象）什么事件（dealloc操作）执行什么操作（置nil）。**

### 动手实现弱引用置nil

对于弱引用不增加引用计数，之前[文章](https://nemocdz.github.io/My-blog/post/objective-c%E5%BC%B1%E5%BC%95%E7%94%A8%E5%AE%B9%E5%99%A8%E5%AE%9E%E7%8E%B0%E6%96%B9%E6%A1%88%E6%80%BB%E7%BB%93/)已经探讨过，现在目的是如何仿照苹果这种设计，去实现置nil的操作。苹果是为了多个弱引用指针指向同一个对象才使用了表，而需要其中一个指针置nil的关键在于，监听dealloc操作。而其实在ARC里，重写dealloc方法就可以，但是怎么样不入侵整个类的dealloc方法呢？这时突破点在于，dealloc中做了什么，有没有办法在dealloc调用的其它方法入手。

而MRC中的dealloc方法实际上做了这些事情：

1. 对自己所有强引用的属性发送release消息
2. 对自己所有强引用的关联对象发送release消息
3. ….
4. 调用``[super dealloc]``

这时我们发现，只要此时关联对象引用计数为1，那么发送release消息后则会调用它的dealloc方法并销毁。

在它dealloc时置我们需要的指针为nil是合适的选择，因为之后的时机指向的对象也会被销毁，销毁之前置nil刚刚好。

为了保证关联对象的引用指针为1，在weak赋值时z只要创建一次就好了。由于我们需要置nil这个操作，关联对象的dealloc跑一个block是灵活性最大的选择了，也就是由关联对象持有一个block，并在weak赋值时顺便告诉这个block里面执行什么。

```objective-c
@interface CDZDeallocObserver : NSObject
@property (nonatomic, copy) void (^block)(void);
@end

@implementation CDZDeallocObserver
- (void)dealloc{
    if (self.block) {
        self.block();
    }
}
@end
```

用分类实现一个setBlock的方法，拓展给NSObject类。

```objective-c
const void *CDZDellocBlockKey = &CDZDellocBlockKey;
@implementation NSObject (DeallocBlock)

- (void)cdz_deallocBlock:(void(^)(void))block{
    CDZDeallocObserver *observer = objc_getAssociatedObject(self, CDZDellocBlockKey);
    if (!observer) {
        observer = [CDZDeallocObserver new];
        objc_setAssociatedObject(self, CDZDellocBlockKey, observer, OBJC_ASSOCIATION_RETAIN);
    }
    observer.block = block;
}
@end
```

### 弱引用容器的拓展

之前我们知道CoreFoundation和MRC法都可以使引用计数不变，那么其实我们只要在Add的时候顺便关联上释放时移除对象即可，因为保留一堆nil指针在容器内并不是一个好的选择。置nil其实更希望是表示这个元素不存在了。

```objective-c
//与CoreFoundation配套的add方法，用unsafe_unretain是不希望被置nil，系统置nil的时机可能比实际dealloc的时机早
- (void)cf_addObject:(id)object{
    [self addObject:object];
    __unsafe_unretained typeof (object) unRetainObject = object;
    __weak typeof(self) weakSelf = self;
    [object cdz_deallocBlock:^{
        if (unRetainObject) {
            [weakSelf removeObject:unRetainObject];
        }
    }];
}
```

```objective-c
//跟mrc配套的
- (void)mrc_addObject:(id)object{
    [self addObject:object];
    __unsafe_unretained typeof (object) unRetainObject = object;
    __weak typeof(self) weakSelf = self;
    [object cdz_deallocBlock:^{
        if (unRetainObject) {
            [weakSelf removeObject:unRetainObject];
        }
    }];
    CFRelease((__bridge CFTypeRef)(object));
}
```

这样才能保证后续在对象销毁时，用容器获取的对象是正确的（置nil或被移除）。

### 最后

更改后的方案[Demo](https://github.com/Nemocdz/OCWeakContainer)已经更新，之前对于内存管理的理解不够深入而导致错误的文章希望不要误导了太多人。

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

##### 参考链接

- [weak 弱引用的实现方式](http://www.desgard.com/weak/)
- [iOS管理对象内存的数据结构以及操作算法--SideTables、RefcountMap、weak_table_t-一](http://www.jianshu.com/p/ef6d9bf8fe59)
- [如何使用 Runtime 给现有的类添加 weak 属性](http://www.jianshu.com/p/ed65d71554d8)



