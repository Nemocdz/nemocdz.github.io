---
title: "Cocoa多线程总结"
date: 2017-06-28T11:31:02+08:00
draft: false
---

个人学习总结笔记

#### 前置知识

##### 线程

程序执行流的最小单元

<!--more-->

* 线程拥有自己的空间：栈，线程局部储存（Thread Local Storage），寄存器
* 线程之间共享：全局变量，堆上的数据，函数的静态变量，代码，文件
* 线程调度中状态：运行，就绪，等待 
* 线程优先级改变方式：1.用户指定 2.等待状态频繁程度 3.长时间不执行 4.抢占和不可抢占

##### 多元信号量

锁的一种，计数N，访问减1，小于0等待

##### 死锁

两个或更多线程阻塞着等待其它处于死锁状态的线程所持有的锁。

产生的四个必要条件

1. 互斥，资源需要等待
2. 请求和保持，请求进程阻塞后不释放自己的资源
3. 不剥夺，无法剥夺他人的资源
4. 环路等待，等待关系形成环

##### 并发

多个任务同时发生，需要被处理

##### 串行与并行

串行：多个任务依次执行

并行：多个任务同时执行

##### 同步与异步

同步：调用者主动等待调用的返回结果

异步：调用者不等待调用的返回结果，调用直接返回，之后被调用者在获得结果后返回结果给调用者

##### 多线程技术

多线程技术使多个cpu可以同时工作，同时处理不同线程的指令。给多核处理器带来并行能力。

优点

1. 某个操作可能陷入长时间等待-等待的线程会进入睡眠状态而无法继续执行
2. 某个操作可能会消耗大量时间-程序和用户的交互会中断
3. 程序的逻辑要求并发操作
4. 单线程无法发挥多核计算机全部能力
5. 资源共享效率提高

缺点

1. 创建消耗cpu和内存
2. 需注意线程安全，资源竞争

### iOS开发中的多线程技术

#### 基础

* iOS启动时运行的线程是主线程
* 主线程占用堆栈1M
* 其余线程是512KB
* 需要主线程操作UIKit，因为UIKit操作不是全都是线程安全的

#### 方案

##### pthread（POSIX threads）

类Unix系统中作为操作系统的通用多线程API

1. 跨平台，可移植
2. C语言
3. 使用难

##### NSThread

1. 面向对象的线程
2. 需要手动管理生命周期，同步，加锁

##### GCD（Grand Central Dispatch）

1. block定义任务，面向过程
2. C语言
3. 使用方便

##### NSOperation+NSOperationQueue

1. 面向对象
2. 指定依赖、取消、暂停恢复、优先级方便
3. 对GCD的封装

#### GCD

CGD=block块执行任务+队列+执行方式

##### 队列

先进先出（FIFO），先追加的任务先执行

* 串行队列（等待之前执行完再执行）
* 并发队列（不需等待之前执行完再执行）
* 主队列（主线程中的串行队列）

##### 执行方式

* 同步
* 异步

主队列操作一定在主线程中，*反之不必然*

*除了主队列外*，同步在当前线程中执行，异步在新建线程中执行

#### API

##### 常用

Dispatch_async/sync

派发同步和异步事件，最常用

Dispatch_after

用于延迟执行，简单定时

Dispatch_get_main_queue/get_global_queue

获取主队列或系统默认的并行队列

Dispatch_queue_create 

新建一个队列

Dispatch_once

只被执行一次+线程安全，常用于实现单例

##### 非常用

Dispatch_queue_set_target_queue

主要用于通过设定目标队列为自建的队列改变为目标队列的优先级，也可指定队列之间的层次，达到自上而下控制队列

Dispatch_group

管理多个队列里事件的执行情况。将队列加入组中，全部结束后追加处理或增加指定时间的任务处理完成情况检测（超时处理）

Dispatch_barrier_async

承上启下，队列里在他之前的执行完成才开始执行他block的，在block执行之后再开始执行队列里后面的。他执行时只有他自己执行，像一个栅栏一样。分隔开之前完成和之后开始的。

Dispatch_apply

按指定次数追加block到对应的队列中

Dispatch_suspen/resume

暂停与恢复

Dispatch I/O

通过GCD分割文件读写，提高速度

Dispatch_semaphore

可设置类似多元信号量中的“0”为x，进行排他性控制。

在操作前后进行信号计数判断，小于x则等待，大于等于则减a并开始执行事件，事件执行完毕后加b（这里的a，b相当于任务对信号量的影响）

#### GCD中的死锁

```objective-c
//主队列中执行以下代码
dispatch_sync(dispatch_get_main_queue(), ^{//操作B
    …//事件A
});
```

这是在一个串行队列（主队列）中执行（往该串行队列派发同步事件A）的操作B，会造成死锁（*反之不必然不死锁*）。

队列先进先出，而串行队列同时执行的任务只有一个。操作B需要等待其派发的block执行完才算完成，派发的同步事件A被追加在队列最后（后进），同步执行需要等待前面执行完才可以执行，而前面的因为操作B没执行完，造成相互等待。

举个例子：面试时要排队（队列），一次只能一个人进面试房间（串行队列），前面的面试完出来后面的人才可以进去面试（同步任务）。A进去房间，面试官说，你的面试内容是叫B来面试，并等他面试完成（派发同步任务Block在串行队列中）。A把B叫来了，B来了就排在了队伍最后面，排队等待轮到他，而A则在房间里等待B面试完。结果就是A等待B面试完等不到，而B则等待前面的人面试完（包括A面试完出来）然后轮到他。两人相互等待，过程就进行不下去了。

```objective-c
dispatch_sync(queueA,^{//操作C
    dispatch_sync(queueB, ^{//操作B
            dispatch_sync(queueA,^{...});//事件A
    })
});
```

在一个串行队列中执行需要等待((往该串行队列派发同步事件A)的操作B)的操作C，会造成死锁。

操作C需要等待操作B，操作B需要等待事件A，事件A需要等待操作C，相互等待。

也就是说，串行队列导致了互斥，block被持有导致了请求和保持，而又无法剥夺，而这时候代码用法导致事件等待关系形成一个环，从而形成相互等待，满足死锁四个条件。

#### GCD的实现原理

[深入了解GCD](https://bestswifter.com/deep-gcd/)

[GCD源码](https://apple.github.io/swift-corelibs-libdispatch/)

#### NSOperation

GCD中Block的面向对象封装

##### 标识状态（只读属性）

* 执行（isExecuted）
* 完成（isFinished）
* 取消（isCancelled）

##### 系统子类

NSBlockOperation和NSInvocationOperation

默认start是同步执行的方式

#### NSOperationQueue

GCD队列的面向对象封装

有Current和Main两个属性可获取当前队列或者主队列

自己生成NSOperationQueue是并发队列，会尽可能为每一个NSOperation创建线程

##### MaxConcureentOperationCount

表示最大并发执行NSOperation的数量

* 设为1时相当于串行队列
* 大于1为并发队列

NSOpertaionQueue调用addOperation方法放入NSOperation对象后相当于*异步执行*

##### 与GCD比较

1. 方便取消，执行前任意取消（GCD在iOS8后有dispatch_block_cancel）
2. 设置Operation间依赖关系
3. KVC观测属性状态
4. 指定优先级
5. 重用NSOperation对象

##### 参考链接

- [Cocoa并发编程](https://hit-alibaba.github.io/interview/iOS/Cocoa-Touch/Multithreading.html)
- [iOS 并发编程之 Operation Queues](http://blog.leichunfeng.com/blog/2015/07/29/ios-concurrency-programming-operation-queues/)
- [iOS多线程编程——GCD与NSOperation总结](https://bestswifter.com/multithreadconclusion/)
- [GCD容易让人迷惑的几个小问题](http://www.jianshu.com/p/ff444d664e51)
- [Objective-C高级编程-iOS与OS X多线程和内存管理](https://book.douban.com/subject/24720270/)
- [程序员的自我修养-链接、装载与库](https://book.douban.com/subject/3652388/)

