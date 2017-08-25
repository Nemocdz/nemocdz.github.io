---
title: "从一道网易面试题浅谈Objective-C线程安全"
date: 2017-08-25T02:46:12+08:00
draft: false
---
今天去网易面试，面试官出了一道面试题，下面代码会发生什么问题？

```objective-c
@property (nonatomic, strong) NSString *target;
//....

dispatch_queue_t queue = dispatch_queue_create("parallel", DISPATCH_QUEUE_CONCURRENT);
for (int i = 0; i < 1000000 ; i++) {
	dispatch_async(queue, ^{
     	self.target = [NSString stringWithFormat:@"ksddkjalkjd%d",i];
	});
}
```

当时我把自定义的队列看成了串行队列，然后回答：“没错呀”。后来一运行崩溃了……

面试后，我就仔细回想，敲了Demo，看看崩溃原因是啥。

正好试试小伙伴给我介绍的调试野指针的方法，XCode7以上才有的[Address Sanitizer](http://www.jianshu.com/p/6b504428e16e)。

打开后发现是经典的EXC_BAD_ACCESS错误，以我浅薄的经验来看，这种一般是对一个已释放的内存的对象再次发送消息出现的。

![屏幕快照 2017-08-25 上午1.55.50](https://ws2.sinaimg.cn/large/006tKfTcly1fivdrg9q8pj31fe022wfi.jpg)

再看看崩溃堆栈

![屏幕快照 2017-08-25 上午1.53.22](https://ws4.sinaimg.cn/large/006tNc79ly1fivdrdngbdj30gw0asaba.jpg)

噢，看来是对已释放的对象再次发送了release信息。

我又留意到，这个对象是Strong修饰的。

那么他的Setter方法在MRC上就相当于

```objective-c
- (void)setTarget:(NSString *)target {
	[target retain];//先保留新值
    [_target release];//再释放旧值
    _target = target;//再进行赋值
}
```

那么什么时候会导致过多调用release呢，因为这是个并行队列+异步。

那么假如队列A执行到步奏2，还没到步骤3时，队列B也执行到步骤2，那么这个对象就会被过度释放，导致向已释放内存对象发送消息而崩溃。

后来我想怎么可以修改这段代码变为不崩溃的呢？

##### 1.使用串行队列

将set方法改成在串行队列中执行就行，这样即使异步，但所有block操作追加在队列最后依次执行。

##### 2. 使用atomic

atomic关键字相当于在setter方法加锁，这样每次执行setter都是线程安全的，但这只是单独针对setter方法而言的狭义的线程安全。

##### 3.使用weak关键字

weak的setter没有保留新值或者保留旧值的操作，所以不会引发重复释放。当然这个时候要看具体情况能否使用weak，可能值并不是所需要的值。

##### 4.使用Tagged Pointer

Tagged Pointer是苹果在64位系统引入的内存技术。简单来说就是对于NSString（内存小于60位的字符串）或NSNumber（小于2^31），64位的指针有8个字节，完全可以直接用这个空间来直接表示值，这样的话其实会将NSString和NSNumber对象由一个指针转换成一个值类型，而值类型的setter和getter又是原子的，从而线程安全。

比如上述代码的字符串改短一些，就不会崩溃了。

从而我们可以总结到，线程安全有以下几种方法：

* 单线程串行访问
* 访问加锁
* 使用不进行额外操作的关键字（weak）
* 使用值类型

然而这只是保证了基本的线程安全（不崩溃），若是需要保证访问出符合预期的数据，则需要采用GCD的barrier或者自己在合适的时机加锁。

##### 参考链接

* [iOS多线程到底不安全在哪里？](http://www.jianshu.com/p/fd81fec31fe7)
* [深入理解Tagged Pointer](http://www.infoq.com/cn/articles/deep-understanding-of-tagged-pointer/)
* [【译】采用Tagged Pointer的字符串](http://www.cocoachina.com/ios/20150918/13449.html)







