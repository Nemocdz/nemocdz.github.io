---
title: "AFNetworking的漂亮细节"
date: 2017-08-16T03:31:13+08:00
draft: false
---

### 写在开头

最近重读了AFNetworking源码，发现很多以前读不懂，也不知道为啥这么写的代码慢慢读懂了。过程中被AFNetworking作者的对细节，舒服，整洁的追求所折服。把一些个人觉得写的漂亮的用法总结下来，本文不在于探讨AFNetworking源码的具体业余实现，尽量从代码本身和设计角度进行总结（源码解析推荐[AFNetworking到底做了什么？](http://www.jianshu.com/p/856f0e26279d)这篇文章）。

#### 1.Dispatch_once方法声明C语言变量方法

感觉很像OC的protery的getter方法，一种C语言懒加载的感觉，static修饰符意味只在该编译单元可见（对应OC就是.m文件），配合单例，只会被执行一次。类似于``if(!object)``的感觉。

如下面例子创建了一个queue的方法，调用后返回是同一个变量。

```c
static dispatch_queue_t url_session_manager_creation_queue() {
    static dispatch_queue_t af_url_session_manager_creation_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_creation_queue = dispatch_queue_create("com.alamofire.networking.session.manager.creation", DISPATCH_QUEUE_SERIAL);
    });

    return af_url_session_manager_creation_queue;
}
```

#### 2.使用反射机制来确定KeyPath

在KVO中，我们一般会观察通过一个属性，而@property其实是一个语法糖，属性=ivar（实例对象）+setter方法+getter方法。而getter方法名就是属性d的名字。利用OC中的反射机制``NSStringFromSelector``方法获取属性的getter方法的字符串，其实就是属性的KeyPath。这样的KeyPath不容易写错，也容易跳转去看属性的定义。

```objective-c
 [progress addObserver:self
            forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                      options:NSKeyValueObservingOptionNew
                      context:NULL];
```

#### 3.用__unused修饰符修饰不用的Delegate中的变量

一般协议delegate声明时，会把delegate弱持有者作为第一个参数传入delegate方法中，可是有时候delegate的实现者并不关心或不区分delegate对象是谁持有的。

```objective-c
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error{
    //......
}
```

#### 4.避开Cocoa类族的坑

在Foudation框架中，某些类其实是类族，比如NSArray，生成的某一个NSArray对象实际上可能是NSArray的子类。所以用Method Swizzling的去Hook一些系统类方法的时候，要注意某些类实际上是子类，甚至不同系统版本继承链和方法实现都不一样（不一定调用了父类的同名方法）。在AFURLSessionManager中为了Hook系统的NSURLSessionTask的resume和suspend的方法实现中加上通知。由于NSURLSessionTask是一个类族，且iOS7和iOS8上Task类的继承链不同，于是有了以下严谨的代码。

```objective-c
if (NSClassFromString(@"NSURLSessionTask")) {
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];//先构建NSURLSession
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];//再通过session对象构建一个task对象
#pragma clang diagnostic pop
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));//获取要交换的方法的实现指针
        Class currentClass = [localDataTask class];//获取真正的子类
        
        while (class_getInstanceMethod(currentClass, @selector(resume))) //检查是否实现需要交换的方法
        {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));//获取实现指针
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));//获取父类的实现指针
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {//如果实现和父类不一样且实现不一样
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];//获取父类继续调用
        }
        
        [localDataTask cancel];
        [session finishTasksAndInvalidate];
}
```

#### 5.增加方法到类中再进行Method Swizzling

同时为了避免直接换带来多次交换把原来方法弄乱的问题，是先将需要换的方法add到需要替换类中（相当于生成一个副本），然后让那个类里面的副本方法去交换，也就是不影响原来拥有这个方法的类里的方法。

```objective-c
+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {
    Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));
    Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));

    if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {
        af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));
    }

    if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {
        af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));
    }
}
```

#### 6.用GCD的同步方法来封装代码块

iOS8以下``dataTaskWithRequest``是生成的task时是并发执行的，造成taskIdentifer偶发不唯一，解决办法是使这个方法串行执行，同时用Dispatch_sync等待结果返回。调用时封装了一个C方法。

```objective-c
 url_session_manager_create_task_safely(^{
        downloadTask = [self.session downloadTaskWithRequest:request];
  });
```

```objective-c
static dispatch_queue_t url_session_manager_creation_queue() {
    static dispatch_queue_t af_url_session_manager_creation_queue;
    static dispatch_once_t onceToken;
 	//创建一个串行队列，只创建一次
    dispatch_once(&onceToken, ^{
        af_url_session_manager_creation_queue = dispatch_queue_create("com.alamofire.networking.session.manager.creation", DISPATCH_QUEUE_SERIAL);
    });
    return af_url_session_manager_creation_queue;
}

static void url_session_manager_create_task_safely(dispatch_block_t block) {
    if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
      	//iOS8以下则调用，同步等待结果
        dispatch_sync(url_session_manager_creation_queue(), block);
    } else {
        block();
    }
}
```

#### 7.重写respondsToSelector更改Delegate实现的判断依据

AFNetworking内部的AFURLSessionManager将所有NSURLSessionDelegate的方法都接管了并转换成外界Set进来的Block实现，其中有一些转换并没有做任何处理，单纯转换成Block。所以是否响应这个Delegate方法其实是block是否存在，于是内部就重写了respondsToSelector。

```objective-c
- (BOOL)respondsToSelector:(SEL)selector {
    if (selector == @selector(URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:)) {
        return self.taskWillPerformHTTPRedirection != nil;
    } else if (selector == @selector(URLSession:dataTask:didReceiveResponse:completionHandler:)) {
        return self.dataTaskDidReceiveResponse != nil;
    } else if (selector == @selector(URLSession:dataTask:willCacheResponse:completionHandler:)) {
        return self.dataTaskWillCacheResponse != nil;
    } else if (selector == @selector(URLSessionDidFinishEventsForBackgroundURLSession:)) {
        return self.didFinishEventsForBackgroundURLSession != nil;
    }

    return [[self class] instancesRespondToSelector:selector];
}
```

#### 8.@dynamic关键字实现默认调用父类setter&getter

当子类重新声明一个父类的属性时，其实默认合成了setter&getter并覆盖了父类的默认实现。对于想子类声明属性却希望默认调用父类属性的setter&getter，可以用@dynamic关键字。

AFHTTPSessionManger是AFURLSessionManger的子类，也声明了securityPolicy属性并重写其setter方法，但希望getter方法调用父类的。

```objective-c
@dynamic securityPolicy;

- (void)setSecurityPolicy:(AFSecurityPolicy *)securityPolicy {
    if (securityPolicy.SSLPinningMode != AFSSLPinningModeNone && ![self.baseURL.scheme isEqualToString:@"https"]) {
        NSString *pinningMode = @"Unknown Pinning Mode";
        switch (securityPolicy.SSLPinningMode) {
            case AFSSLPinningModeNone:        pinningMode = @"AFSSLPinningModeNone"; break;
            case AFSSLPinningModeCertificate: pinningMode = @"AFSSLPinningModeCertificate"; break;
            case AFSSLPinningModePublicKey:   pinningMode = @"AFSSLPinningModePublicKey"; break;
        }
        NSString *reason = [NSString stringWithFormat:@"A security policy configured with `%@` can only be applied on a manager with a secure base URL (i.e. https)", pinningMode];
        @throw [NSException exceptionWithName:@"Invalid Security Policy" reason:reason userInfo:nil];
    }
    [super setSecurityPolicy:securityPolicy];
}
```

#### 9.KVO用Context区分父类与子类

KVO中的Api``- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context``总是在dealloc方法中配合``- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath context:(nullable void *)context``成对使用来移除观察者。但如果父类和子类都同时观察一个keyPath，那么容易导致addObserver和removeObserver的个数不匹配（子类未调用父类的addObserver方法但调用了父类的dealloc），导致重复调用remove同一个keyPath而Crash。所以加上context作为类别唯一标识才是比较安全的做法。

```objective-c
- (void)dealloc {
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self removeObserver:self forKeyPath:keyPath context:AFHTTPRequestSerializerObserverContext];
        }
    }
}
```

#### 10.用GCD实现同步属性

实现某个属性值setter方法和getter方法的同步除了用NSLock或者@synchronized关键字加锁外，可以使用在并发队列里setter配合dispatch_barrier_async加上getter配合dispatch_sync实现。这样读取是同步并发的，写入是在没有读取都完成后，同步执行的，并且性能比加锁更好。AFHTTPRequestSerializer里的HTTPHeaderField就是这样的。

```objective-c
- (void)setValue:(NSString *)value
forHTTPHeaderField:(NSString *)field
{
    dispatch_barrier_async(self.requestHeaderModificationQueue, ^{
        [self.mutableHTTPRequestHeaders setValue:value forKey:field];
    });
}

- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    NSString __block *value;
    dispatch_sync(self.requestHeaderModificationQueue, ^{
        value = [self.mutableHTTPRequestHeaders valueForKey:field];
    });
    return value;
}
```

#### 11.对NSStream类的操作

AFHTTPRequestSerizlizer进行Multipart协议支持时，使用了NSStream对象。

一般使用方法是

```objective-c
//创建流对象
NSInputStream *stream = [[NSInputStream alloc] initWithData:[NSData data]];

//放入Runloop中
[stream scheduleInRunLoop:NSRunLoop.currentRunLoop forMode:NSDefaultRunLoopMode];

//打开流
[stream open];

//.......

//从Runloop中移除
[stream removeFromRunLoop:NSRunLoop.currentRunLoop forMode:NSDefaultRunLoopMode];

//关闭流
[stream close];
```

结果发现AFNetworking在析构NSStream时，没有调用removeFromRunLoop，仅仅调用了close，我一开始还以为漏了，结果后来书写Demo验证发现其引用计数的变化时如下的。

```objective-c
//放入Runloop中-----引用计数+2
[stream scheduleInRunLoop:NSRunLoop.currentRunLoop forMode:NSDefaultRunLoopMode];

//打开流-----引用计数不变
[stream open];

//.......

//从Runloop中移除-----引用计数-2
[stream removeFromRunLoop:NSRunLoop.currentRunLoop forMode:NSDefaultRunLoopMode];

//关闭流-----引用计数-2
[stream close];
```

也就是说open和close对引用计数的影响不是一对的，从一定程度上解释只调用close也可以达到removeFromRunLoop的原因，个人猜测close和removeFromRunLoop调用任意一个都可以。顺便提一句AFMultipartBodyStream还对NSStreamStauts一些只读属性改成读写的，并自定义了所有流的方法。

#### 12.Category中实现属性懒加载

使用了OC的关联对象，先获取判断是否为空，不然就生成并关联上。

```objective-c
- (AFRefreshControlNotificationObserver *)af_notificationObserver {
    AFRefreshControlNotificationObserver *notificationObserver = objc_getAssociatedObject(self, @selector(af_notificationObserver));
    if (notificationObserver == nil) {
        notificationObserver = [[AFRefreshControlNotificationObserver alloc] initWithActivityRefreshControl:self];
        objc_setAssociatedObject(self, @selector(af_notificationObserver), notificationObserver, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return notificationObserver;
}
```

#### 13.头文件明确声明Nonull和Nullable

自从支持Swift后，和Swift中的?和!对应，OC引入了Nonull和Nullable关键字，当然对每个方法参数和属性声明关键字是很大工作量的，这时我们可以用一对系统宏包含在最前和最后，中间的默认关键字就是Nonull了，这时候针对Nullable的参数或属性进行补充就可以了。

```objective-c
//.h
NS_ASSUME_NONNULL_BEGIN

//...
@property (nonatomic, strong, nullable) id <AFImageRequestCache> imageCache;
//...

NS_ASSUME_NONNULL_END
```

### 最后

AFNetworking是一份写得十分严谨漂亮的源码，其中对KVO&KVC，GCD，Block，关联对象的运用十分巧妙且准确，同时接口的封装，代码的划分也很恰当。阅读之后对于代码规范又有了新的理解。





