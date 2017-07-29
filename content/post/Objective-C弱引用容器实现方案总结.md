---
title: "Objective-C弱引用容器实现方案总结"
date: 2017-07-23T04:55:00+08:00
draft: false
---

在OC中Foundation框架中的常用容器类（NSSet，NSDictionary，NSArray）及其可变子类在加入元素时，均会对元素进行强引用。有的时候（比如持有多个Delegate对象时），希望有对应的弱引用容器使用。

#### 实现思路

根据加入元素这个操作，很容易发现实现方案从容器本身、加入对象这个行为、对象本身三个方向入手。

### 容器本身支持弱引用对象

#### 1.Foundation框架支持弱引用的容器类

* NSHashTable-对应NSMutebleSet
* NSMapTable-对应NSMutebleDictionary
* NSPointerArray-对应NSMutebleArray

本身都是可变的，没有不可变父类。其共同点是addObject方法参数声明为nullable即不用判断是否为nil来避免崩溃，且都拥有初始化方法``- (instancetype)initWithOptions:(NSPointerFunctionsOptions)options``而参数options代表其所支持的放入对象的指针管理选项。根据苹果官方文档显示，这些枚举值被分为三类。

* Memory Options（内存语义管理选项）

  ```objective-c
  NSPointerFunctionsStrongMemory // 和strong一样，默认
  NSPointerFunctionsOpaqueMemory // 在指针去除时不做任何动作
  NSPointerFunctionsMallocMemory // 去除时调用free() , 加入时calloc()
  NSPointerFunctionsMachVirtualMemory //使用可执行文件的虚拟内存
  NSPointerFunctionsWeakMemory //和weak一样         
  ```

* Personaility Options(对象处理选项-如何进行哈希算法，判定等同性，描述)

  ```objective-c
  NSPointerFunctionsObjectPersonality //使用NSObject的hash、isEqual、description，默认
  NSPointerFunctionsOpaquePersonality //使用偏移后指针，进行hash和直接比较等同性
  NSPointerFunctionsObjectPointerPersonality //和上一个相同，多了description方法	  
  NSPointerFunctionsCStringPersonality //使用c字符串的hash和strcmp比较，%s作为decription
  NSPointerFunctionsStructPersonality // 使用内存的hash和memcmp 
  NSPointerFunctionsIntegerPersonality //使用偏移量作为hash和等同性判断
  ```

* Copy Options（对象拷贝选项）

  ```objective-c
  NSPointerFunctionsCopyIn//通过NSCopying方法，复制后存入
  ```

另外，文档还强调，**每种类别的选项互斥，只能每种类别选择一个**。

当然，这三种容器还提供了快捷实例化类方法，含有weakObjects字样。

其中NSPointerArray稍微和其他两个有点特殊。

1. 不支持OC的轻量级泛型（未被声明为<Object Type>）
2. 猜测和1有关，addPointer方法（相当于NSMuteableArray的addObject方法）参数类型是void*指针，所以需要(__bridge void *）转换

这个方案应该是最简单的了，毕竟是苹果Foundation框架里封装的高级对象。但是其缺点是性能相比对应强引用容器在性能上有所欠缺，根据Objc.io文章中的测试结果，**NSHashTable的addObject和NSPointerArray的addPointer**性能差距尤其严重，使用时需要注意。

![屏幕快照 2017-07-23 上午3.04.13](https://ws1.sinaimg.cn/large/006tKfTcly1fht96ofpjqj30ig0cqwfz.jpg)

![屏幕快照 2017-07-23 上午3.04.21](https://ws1.sinaimg.cn/large/006tKfTcly1fht96ud6zxj30h1084mxy.jpg)

![屏幕快照 2017-07-23 上午3.04.29](https://ws4.sinaimg.cn/large/006tKfTcly1fht970xifdj30er06bgm6.jpg)



#### 2.用CFFoundation框架里对应容器类自定义内存管理选项

在初始化时，使用CFFoundation里的容器类实现后转成Foundation对象

以NSMutableSet为例，可以增加一个新的分类，自定义一个实例化方法

```objective-c
+ (instancetype)cdz_weakSet{
    CFSetCallBacks callbacks = {0, NULL, NULL, CFCopyDescription, CFEqual, CFHash};
    return (NSMutableSet *)CFBridgingRelease(CFSetCreateMutable(0, 0, &callbacks));
}
```

建立容器类的步骤

1. 生成CallBacks结构体，结构体中的对象其实就是一些描述容器的选项

   同样以CFSetCallBacks为例，其结构体定义如下

   ```objective-c
   typedef struct {
       CFIndex				version;
       CFSetRetainCallBack			retain;
       CFSetReleaseCallBack		release;
       CFSetCopyDescriptionCallBack	copyDescription;
       CFSetEqualCallBack			equal;
       CFSetHashCallBack			hash;
   } CFSetCallBacks;
   ```

   version暂时不管它，看下剩下变量类型名字，是否感觉比较熟悉

   其实和之前NSPointerFunctionsOptions类似，还是分三种，retain和release是内存管理，equal和hash和是对象处理，copyDescription是复制处理。

   所以我们就需要改变retain和release就好。类型是xxxCallBack那么说明应该这个类型是一个block，而去当这些操作执行执行对应的block。文档对retain和release这两个block的描述也是这样说的。那么如果要实现引用计数增减，就应该在这两个block里实现，所以我猜测Foundation框架里的容器类retain，应该就在这个两个block里实现。那么我们只需要block值为NULL，在retain和release操作里，就不会引用计数的变化，从而不持有对象。

2. 使用CreatMutable生成可变容器
   这时传入之前的自定义callBacks，也可以传入capacity之类的（默认为0就好），转换为Foundation对象


之后我们只要有这个实例化方法生成的容器类，其它就和正常使用一样了。

### 加入容器时不引用

方式很简单，在addObject后调用一次release减少引用技术。

release方法有两种

1. 将分类.m文件将编译选项切换为MRC（在Build Phases中的Compile Sources中加入编译标记-fno-objc-arc），直接使用[object release]
2. 在ARC中，使用CFFoundation的release方法CFRelease

例子中使用第二种，在分类中新增一个方法

```objective-c
- (void)cdz_weakAddObject:(NSObject *)object{
    [self addObject:object];
    CFRelease((__bridge CFTypeRef)(object));
}
```

**注意！！**

在使用这种方法后，这个容器将变得极其不安全，必须搭配一整套API来操作容器，包括add/remove/objectOfxxx，而不搭配使用的话，将可能将引用计数变得混乱，甚至导致崩溃。

而最起码的，在容器会被销毁时，需要搭配一个特殊的removeAllObjects去将引用计数进行修正。否则因为在addObject时没有增加引用计数，则在容器销毁时，容器内部的对象的引用计数已经将至0，对象被销毁（而不是置nil），内存被回收（标记为可用）。此时当容器在销毁时，会遍历容器里所有对象并对他们发送release消息，代表不再持有该对象（引用计数减1），会向一个已被销毁的对象发送消息，导致崩溃。

这也是MRC时代遇见比较多的过度释放（over release）问题。

可以在Xcode开启NSZombie调试选项（释放时转变为NSZombie并记录收到的消息）进行追踪过渡释放。

**笔者不推荐这种方式实现，因为在使用时必须十分小心，而团队维护也容易不小心和系统Api混合使用**

### 使对象变得可被弱引用

思路就是用一个别的对象A持有这个原来元素的弱引用，在容器类存入对象A。抛砖引玉，提供两种方法

#### 1.使用NSValue

使用NSValue的valueWithNonretainedObject转化为NSValue对象存入容器。

这个方法会弱引用这个object。

取的时候用value的nonretainedObjectValue属性

```objective-c
NSValue *weakObject = [NSValue valueWithNonretainedObject:object];
weakObject.nonretainedObjectValue;
```

#### 2.使用block

先对object加上weak修饰符，在包在block中就可

使用时可搭配以下简便方法

```objective-c
typedef id(^CDZWeakObjectBlock)();
CDZWeakObjectBlock blockOfObjcet (id object) {
    __weak id weakObjcet = object;
    return ^{
        return weakObjcet;
    };
}

id objectOfBlock (CDZWeakObjectBlock block) {
    if (block) {
        return block();
    }
    else {
        return nil;
    }
}
```

当然也可以实现自定义对象进行持有，而缺点就是，**没办法使用容器的泛型进行约束和警告**，差异值被抹去了。

### 最终方案

自己实现一个容器类，哈哈

这里提供自己浅显的思路，参考系统API，容器类关心下面这几个点

* 容器里对象的内存管理
* Hash算法，等同性算法，描述
* 可拷贝
* 对象类型，泛型支持

### 总结

##### Foundation中的弱引用容器类NSHashTable/NSMaptable/NSPointerArray

* 使用简单，功能强大，高层级封装
* 性能有差距

##### CFFoundation实现容器

* 实现稍复杂
* 没有感觉有什么特别大缺点，使用方便需要耦合分类算一个吧

##### 操作容器时实现引用计数修正

* 使用简单
* 实现繁琐（几乎都要实现对应的），需要了解MRC，注意管理引用计数，和系统API混用时容易出问题

##### 使用新对象持有弱引用对象

* 低耦合，按需选择
* 失去容器类泛型修饰

### 最后

所有方案的代码可在[Demo](https://github.com/Nemocdz/OCWeakContainer)中找到

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

##### 参考链接

- [NSHashTable & NSMapTable](http://nshipster.com/nshashtable-and-nsmaptable/)
- [The Foundation Collection Classes](https://www.objc.io/issues/7-foundation/collections/)
- [NSValue](http://nshipster.cn/nsvalue/)
- [NSArray of weak references (__unsafe_unretained) to objects under ARC](https://stackoverflow.com/questions/9336288/nsarray-of-weak-references-unsafe-unretained-to-objects-under-arc)
- [Apple Developer Document - NSPointerFunctionsOptions](https://developer.apple.com/documentation/foundation/nspointerfunctionsoptions?language=objc)
- [Apple Developer Document - CFSetCallBacks](https://developer.apple.com/documentation/corefoundation/cfsetcallbacks)