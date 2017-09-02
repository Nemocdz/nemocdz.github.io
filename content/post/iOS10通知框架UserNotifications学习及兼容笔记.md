---
title: "iOS10通知框架UserNotifications学习及兼容笔记"
date: 2017-07-28T18:18:13+08:00
draft: false
---

在iOS10上，苹果将原来散落在UIKit中各处的用户通知相关的代码进行重构，剥离，打造了一个全新的通知框架-UserNotifications。笔者最近在开发公司通知相关的需求，跟着WWDC2016的视频和官方文档，学习了一下新框架。同时，在学习过程中，和老框架对应Api进行对比，有了个人的感受和看法。

<!--more-->

首先，对于通知框架，其框架功能包括以下四类

* 申请权限/注册配置
* 发送本地通知
* 展示和响应本地/远程通知
* App Extension

 在UserNotifications框架中，最核心的类是**UNUserNotificationCenter**,这个类的是这三项功能的管理类，通过注入到currentNotificationCenter进行对消息的管理。而在之前，大部分操作的管理者UIApplication的单例。

### 1.申请权限/注册配置

在新框架中，将申请权限和注册配置拆分为两个Api，使得职责更加分明。先看旧框架的实现。

```objective-c
UIUserNotificationType types = ...//通知可显示的样式

UIUserNotificationSettings *settting = [UIUserNotificationSettings settingsForTypes:types categories:nil]; //将样式和Category一起生成配置

[[UIApplication sharedApplication] registerUserNotificationSettings:settting];//用这个配置注册
```

Category的概念，是一个通知的种类。对同种类的通知，可以规定对应的一组按钮操作，同时，iOS10以上的Rich Notification（可以展示图片，自定义视图）的区分读也是通过category来区分。

而在老的通知框架中，上面第二行代码中可以看到，category和展示样式一同绑定起来用于申请权限。实际上，Category的概念用于区分接收消息的种类，也就是其实这个概念属于接收消息后进行自定义处理的使用者，也就是App内部开发。而对于申请权限，其实最终是向App外部的交互，也就是App用户关心的。而新框架进行拆分后，就更加自由和职责区分了。

UserNotifications中请求权限的用法

```objective-c
[[UNUserNotificationCenter currentNotificationCenter] requestAuthorizationWithOptions:... completionHandler:^(BOOL granted, NSError * _Nullable error) {
        if (granted) {
			//...如果被授权了
        }
}];
```

同时，增加一个callback将授权后的状态返回，就像分散型网络请求一般。

对于远程通知，还需要一个获取用户token的操作，并使用这个token进行APNs推送。但是奇怪的是，对于申请token这个操作，新框架中却没有对应的Api，沿用旧的Api。

```objective-c
[[UIApplication sharedApplication] registerForRemoteNotifications];

//AppDelegate
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken{
  ....
}
```

获取操作被放在AppDelegate的回调中，如果按照新框架的设计，应该会被设计成一个带callBack的Api。所以笔者猜测沿用旧设计的原因是，在以后，获取token的操作不一定由发起请求后产生，也许可能会有多种情况下产生获取操作，也可能token会过期可以进行更新，重置操作。所以delegate的集中式管理更加拥有拓展性，当然也有可能这个Api忘记迁移哈哈。

#### 配置Category并注册

对于Category的作用，上文已经有介绍，而Category中目前主要的功能，是对应一组按钮操作。

Category的使用分四步

1. 创建Category，设置identifer，配置一组按钮（可无）
2. 将Category注册到UIApplication(旧)/UNNotificationCenter(新)中
3. 发送本地或远程消息时需要按钮或自定义视图时带上对应的categoryIdentifer
4. 收到消息进行响应时通过消息中携带的categoryIdentifer进行分类，可以指定不同的操作

旧框架中按钮动作对象UIUserNotificationAction和新的UNNotificationAction差异性不大，主要关心的是

* 按钮上显示的文字
* 按钮的identifer，用于区分按钮
* 是否需要跳转到前台，唤起主App

而差异性在于

* 旧框架分为可变对象和不可变对象，新框架只有不可变对象加上实例化方法
* 新框架中，对于可输入操作的按钮，拆分成一个子类，可以使输入操作自定义性和拓展性更强

新建一个按钮也很简单

```objective-c
UNNotificationAction *callDriverAction = [UNNotificationAction actionWithIdentifier:@"xxx" title:@"呼叫司机" options:UNNotificationActionOptionForeground];
```

options中可以选择是否需要解锁后才能操作、按钮颜色是否为红色（代表操作有破坏性）、是否打开App。

然后将整组按钮加入UNNotificationCategory（新）或UIUserNotificationCategory（旧）中。

这两个对象差异性在于

* 旧框架分为可变对象和不可变对象，新框架只有不可变对象加上实例化方法
* 旧框架按钮有两种使用环境，所以需要设置UIUserNotificationActionContext
* 新框架支持配置按钮的响应是否发送到UNNotificationCenter的Delegate或CarPlay中
* 新框架支持Intent框架（SiriKit使用到）

##### UIUserNotificationActionContext

* UIUserNotificationActionContextDefault-对应iOS10以下的的“提醒”样式，是一个弹框
* UIUserNotificationActionContextMinimal-对应iOS10以下的“横幅”样式，最多支持两个按钮横向并排，多于两个按钮会取前两个

为什么这个在新的框架中被去掉了呢？

* iOS10上，提醒样式不再是一个弹框，而是和“横幅”统一，只是不会主动往上收起
* iOS10上，横幅的按钮不再是横向并排，而是竖着排放，也不会限制个数

之后调用setNotificationCategories方法注册即可

```objective-c
[[UNUserNotificationCenter currentNotificationCenter]setNotificationCategories:[NSSet setWithObject:self.category]];
```

**在运行app时，原来的set里的categories并不会清空，所以需要将整个Set传进去作为参数，这样会把原来的set完整替换成新的set。**每次调用时，整个set替换原来的set。

同时，对于通知设置和Category设置，拆分了两个Api去获取当前的设置。

```objective-c
- (void)getNotificationCategoriesWithCompletionHandler:(void(^)(NSSet<UNNotificationCategory *> *categories))completionHandler;

- (void)getNotificationSettingsWithCompletionHandler:(void(^)(UNNotificationSettings *settings))completionHandler;
```

而对于按钮的操作响应，将在第三部分响应通知中介绍。

### 2.发送本地通知

在iOS10以前，本地通知的类用的是UILocalNotification。而在UserNotifications框架中，将本地通知和远程通知统一起来，然后将通知拆分成request=content（内容）+trigger（触发器）的模式，十分像网络请求的思路。这样设计的好处是对于远程通知和本地通知对于响应的处理得到统一，同时也不会像以前一样，将本地特有的功能随意堆砌在本地通知类中，而是通过差异化配置类去进行注入。

#### UNNotificationContent

新的通知内容类，有可变子类。同时有以下新功能

* 支持一个UNNotificationAttachment（附件）数组，附件用一个identifer+文件路径构成，可携带视频/图片等，而这些内容也为iOS10的RichNotification自定义视图提供了素材
* 支持Title+Subtitle，Apns对应字段也同步支持
* 通知声音有了特定类UNNotificationSound进行管理，以后拓展性，自由度更好了
* threadIdentifer(主要用于Extesion Content中，在最后一节App Extension中会介绍)

#### UNNotificationTrigger

而之前散落成一个个不同类型的触发相关属性，也被汇总成了UNNotification的子类。有以下几个类

* UNPushNotificationTrigger-这个类代表这条消息是由APNs推送过来的，也就是这个trigger是否是这个类是**区分本地通知和远程通知**的标志，对于做多系统b版本兼容时，很有帮助
* UNTimeIntervalNotificationTrigger-根据相隔时间触发，就和计时器一样，**注意timeInterval要大于0，且希望repeats的话，需要timeInterval大于60**
* UNCalendarNotificationTrigger-根据日历时间触发
* UNLocationNotificationTrigger-根据定位在某个位置触发

#### UNNotificationRequest

request除了包含content和trigger外，还有一个非常主要的属性，那就是**identifer**。

有了Identifer，可以实现通知的更新和移除。

把通知分成待展示（已投递，但未触发或未展示）和已经展示过进入用户通知中心的两种。远程通知在后台收到会立即展示。也就是一共有四种情况

* 更新未展示的通知（本地&远程前台）
* 更新已展示的通知（本地&远程）
* 取消未展示的通知（本地）
* 取消已展示的通知（本地）

而标识同一通知的标志，就是通知的identifer。

而发送/更新通知用的是同一个Api，就是将request添加到UNNotificationCenter里，同时也有一个callback回调状态

```objective-c
[[UNUserNotificationCenter currentNotificationCenter] addNotificationRequest:[UNNotificationRequest requestWithIdentifier:sameIdentifer content:content trigger:trigger] withCompletionHandler:^(NSError * _Nullable error) {
		...
}];
```

而取消或移除未展示和已展示的通知，分开了两组Api方便根据情况选择，同时也有get方法获取当前已展示/未展示的队列里的所有通知，而且和添加/更新不同，移除只需要一个identifer，且可以同时传入一组identifer，一次性移除多个

```objective-c
- (void)getPendingNotificationRequestsWithCompletionHandler:(void(^)(NSArray<UNNotificationRequest *> *requests))completionHandler;//获取未展示通知
- (void)removePendingNotificationRequestsWithIdentifiers:(NSArray<NSString *> *)identifiers;//取消未展示通知
- (void)removeAllPendingNotificationRequests;//取消所有未展示通知队列里的通知

- (void)getDeliveredNotificationsWithCompletionHandler:(void(^)(NSArray<UNNotification *> *notifications))completionHandler;//获取已展示通知
- (void)removeDeliveredNotificationsWithIdentifiers:(NSArray<NSString *> *)identifiers;//从通知中心中移除已展示通知
- (void)removeAllDeliveredNotifications;//移除所有通知中心里的通知
```

### 3.展示和响应

在旧框架里面，这部分Api是最为混乱的部分。一共有1..2.....7个delegate......而且之中有的还有取代关系，也就是有其中一个另一个不执行......

```objective-c
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo;//前台收到远程通知

- (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification;//前台收到本地通知

- (void)application:(UIApplication *)application handleActionWithIdentifier:(nullable NSString *)identifier forLocalNotification:(UILocalNotification *)notification completionHandler:(void(^)())completionHandler;//iOS9之前本地通知点击按钮后

- (void)application:(UIApplication *)application handleActionWithIdentifier:(nullable NSString *)identifier forRemoteNotification:(NSDictionary *)userInfo withResponseInfo:(NSDictionary *)responseInfo completionHandler:(void(^)())completionHandler;//iOS9远程通知点击按钮后

- (void)application:(UIApplication *)application handleActionWithIdentifier:(nullable NSString *)identifier forRemoteNotification:(NSDictionary *)userInfo completionHandler:(void(^)())completionHandler;//iOS9之前远程通知点击按钮后

- (void)application:(UIApplication *)application handleActionWithIdentifier:(nullable NSString *)identifier forLocalNotification:(UILocalNotification *)notification withResponseInfo:(NSDictionary *)responseInfo completionHandler:(void(^)())completionHandler;//iOS9本地通知点击按钮

- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler;//iOS7以上收到远程通知调用(未被废弃)
```

这么乱，让我重新分类。

* 收到通知
* 点击通知本身
* 点击通知的自定义按钮

#### 收到通知

在触发本地通知时，App并不会被唤醒，所以本地只有前台时才有回调。

* 本地+前台-``didReceiveLocalNotification``
* 远程+前台-``didReceiveRemoteNotification``和``didReceiveRemoteNotification:fetchCompletionHandler``若有后面则只执行后面那个
* 远程+后台-唤醒并执行``didReceiveRemoteNotification:fetchCompletionHandler``

#### 点击通知本身

* 本地+App存活-``didReceiveLocalNotification``

* 本地+App未存活-``-(BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions``，其中lanchOptions中UIApplicationLaunchOptionsLocalNotificationKey为UILocalNotification对象

* 远程+App存活-``didReceiveRemoteNotification``和``didReceiveRemoteNotification:fetchCompletionHandler``若有后面则只执行后面那个

* 远程+App未存活-``didReceiveRemoteNotification:fetchCompletionHandler``有则执行，无则``-(BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions``，其中lanchOptions中UIApplicationLaunchOptionsRemoteNotificationKey

  为通知的userInfo

#### 点击自定义按钮

* iOS9``handleActionWithIdentifier ``，有responseInfo后缀的优先调，无则调用无后缀的
* iOS8调用无后缀的

整理完感觉就是，乱，还涉及到lauchOptions等无关api，api之间还会覆盖，还有一些不是收到和响应都会调。

而新框架单纯拆分为收到消息+点击事件响应，且不再在Api层面区分远程与本地通知，使对待通知的路径变得统一。

#### 通知的收到

在前台的时候，收到不管是本地还是远程通知都会被新的delegate接管。以前的通知框架，在前台收到通知时，默认是不展示（没有横幅等）的。而新delegate可以在前台收到通知后，展示前，做一些处理，包括以什么形式展示，已经做一些自定义操作。而这个Delegate只有App在前台才会执行。因为在后台App收到通知时，通知不会唤醒App（不是唤起），而App可能是被杀死的，所以这个Delegate没有被赋值，所以统一只在App在前台时才执行。而将新的Api对应旧的Api转换起来的话，就是下面的代码。

```objective-c
- (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler{
    BOOL isRemote = NO;
    if ([notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
        isRemote = YES;
    }
    UILocalNotification *localNotification;//从新的转换为旧的本地通知
    
     if (!isRemote && [[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:didReceiveLocalNotification:)]) {
        [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] didReceiveLocalNotification:localNotification];
    }
    else if (isRemote) {
        if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:didReceiveRemoteNotification:fetchCompletionHandler:)]) {
            [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] didReceiveRemoteNotification:notification.request.content.userInfo fetchCompletionHandler:^(UIBackgroundFetchResult result) {}];
        }
        else if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:didReceiveRemoteNotification:)]){
            [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] didReceiveRemoteNotification:notification.request.content.userInfo];
        }
    }

    
    completionHandler(UNNotificationPresentationOptionBadge|UNNotificationPresentationOptionAlert);//决定前台展示形式
}
```

而CompletionHandler里传入希望通知支持的类型（横幅、红点、声音），在自定义操作完成后执行这个block就可。当然可以在这里为不同消息定制不同的支持类型。

#### 点击通知或按钮的操作响应

新框架中，点击操作和收到通知的Api的响应终于被区分开了。而且点击操作也包含点击通知本身，以及有了点清除关闭通知的事件响应。拆分点击和收到的Api是很好的，因为点击通知和按钮时，实际上会将app唤起到前台，一般这时候需要进行页面跳转，界面状态恢复等等，这些在收到通知时是没有必要的。

新的Api和就的转换的话就是下面这样

```objective-c
- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler{
    BOOL isRemote = NO;
    if ([response.notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
        isRemote = YES;
    }
    
    UILocalNotification *localNotification;//从新的转换为旧的本地通知

    
    if ([response.actionIdentifier isEqualToString:UNNotificationDefaultActionIdentifier] || [response.actionIdentifier isEqualToString:UNNotificationDismissActionIdentifier]) {
        if (!isRemote && [[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:didReceiveLocalNotification:)]) {
            [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] didReceiveLocalNotification:localNotification];
        }
        else if (isRemote) {
            if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:didReceiveRemoteNotification:fetchCompletionHandler:)]) {
                [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] didReceiveRemoteNotification:response.notification.request.content.userInfo fetchCompletionHandler:^(UIBackgroundFetchResult result) {}];
            }
            else if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:didReceiveRemoteNotification:)]){
                [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] didReceiveRemoteNotification:response.notification.request.content.userInfo];
            }
        }
    }
    else{
        if (!isRemote) {
            if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:handleActionWithIdentifier:forLocalNotification:withResponseInfo:completionHandler:)]) {
                [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] handleActionWithIdentifier:response.actionIdentifier forLocalNotification:localNotification withResponseInfo:@{} completionHandler:^{}];
            }
            else if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:handleActionWithIdentifier:forLocalNotification:completionHandler:)]){
                [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] handleActionWithIdentifier:response.actionIdentifier forLocalNotification:localNotification completionHandler:^{}];
            }
        }
        else{
            if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:handleActionWithIdentifier:forRemoteNotification:withResponseInfo:completionHandler:)]) {
                [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] handleActionWithIdentifier:response.actionIdentifier forRemoteNotification:response.notification.request.content.userInfo withResponseInfo:@{} completionHandler:^{}];
            }
            else if ([[UIApplication sharedApplication].delegate respondsToSelector:@selector(application:handleActionWithIdentifier:forRemoteNotification:completionHandler:)]){
                [[UIApplication sharedApplication].delegate application:[UIApplication sharedApplication] handleActionWithIdentifier:response.actionIdentifier forRemoteNotification:response.notification.request.content.userInfo completionHandler:^{}];
            }
            
        }
    }

    }
    completionHandler();
}
```

当实现了新框架这两个Delegate后，旧框架的6个Delegate将不会被执行，所以除非App只支持iOS10，尽量按上面的代码进行兼容。

**注意-(void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler;**这个方法并未随着旧框架被废弃，还是正常使用。

### 4.App Extension

关于新通知框架的Extension，有以下两个

![屏幕快照 2017-07-26 下午3.16.09](https://ws4.sinaimg.cn/large/006tKfTcly1fhxb6j1b04j308403c0sr.jpg)



#### NotificationService Extension

这个Extension允许我们在远程通知收到前做一些修改。因为之前App内UNNotificaionCenter的Delegte并不能在App不存活情况下执行，所以有了这个Extension来提供这样的功能。新建NotificationService的target后，发现系统会自动生成了UNNotificationServiceExtension的子类。其中重写方法的模板，系统也生成好了。

```objective-c
@interface NotificationService ()
@property (nonatomic, strong) void (^contentHandler)(UNNotificationContent *contentToDeliver);
@property (nonatomic, strong) UNMutableNotificationContent *bestAttemptContent;
@end

@implementation NotificationService

- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request withContentHandler:(void (^)(UNNotificationContent * _Nonnull))contentHandler {
    self.contentHandler = contentHandler;
    self.bestAttemptContent = [request.content mutableCopy];
    
    self.contentHandler(self.bestAttemptContent);
}

- (void)serviceExtensionTimeWillExpire {

    self.contentHandler(self.bestAttemptContent);
}

@end
```

第一个方法在收到远程通知时，可以拦截通知并修改通知内容。先用一个block保存当时的上下文，再修改完后再调用block再修改完新的通知内容回调回去。

第二个方法是在修改通知时机将要结束时调用，这时候会强制执行block了。

在推送 payload 中增加一个 `mutable-content` 值为 1 来启用这个Extension，暂时还不支持本地通知。

#### NotificationContent Extension

这个就是实现Rich Notification的拓展。新建这个Target后，可以发现自动生成了一个ViewController并遵循了UserNotificationsUI框架的UNNotificationContentExtension协议。又是一个新框架，看来之后在通知视图上的功能将会变得更强大。

```objective-c
- (void)didReceiveNotification:(UNNotification *)notification {
	//收到通知后的操作
}

- (void)didReceiveNotificationResponse:(UNNotificationResponse *)response completionHandler:(void (^)(UNNotificationContentExtensionResponseOption option))completion{
   //收到通知响应，如按钮，可以这个方法里拦截，并自定义操作，再决定是否将按钮放行至主app
}
```

这个协议里面只有上面两个方法。第一个方法用于在展开Rich视图时触发，进行根据数据去改变视图。就如平时收到网络请求后根据网络请求返回的响应改变视图元素。当打开RIch视图后，这个方法还有两个时机会触发。

1. Request的Identifer一致，相当于更新整个通知，同时视图也重新加载
2. 还记得之前说的Content的threadIdentifer么，这个相同时，这个方法也会重新调用，也就是这个是用来标识同一个流程的的通知，当然，通知还是两条，只是视图从通知A状态到通知B状态了

而第二个方法，实在通知响应时，比如按钮响应时，先在这个Extension里拦截，可以进行响应修改，视图改变等操作，然后通过CompletionHandler的UNNotificationContentExtensionResponseOption来选择放行这些响应。其中UNNotificationContentExtensionResponseOptionDismissAndForwardAction才可以将响应放行到主app里UNNotificationCenter的delegate方法中。

同时这个Extension也有自己的Storyboard和Info.plist。

![](https://ws4.sinaimg.cn/large/006tKfTcly1fi19jl5hduj30gz05pdgk.jpg)

DefaultContenHidden代表是否在Rich状态下还显示原来的body文字

Category代表此类categoryIdentifer会触发这个ViewController，可以是一个Array，多个同时支持这个ViewController。

SizeRatio代表高/宽比，其中宽固定为屏幕宽度，也就是用这个来调整视图的高度。

### iOS11New

都是一些小改动，增加一些细分化Api等，可以在苹果官方文档找到。

https://developer.apple.com/documentation/usernotifications?changes=latest_major&language=objc

### 总结

#### iOS10新通知框架和旧框架差异性

| UILocalNotification                      | UNNotificationRequest                    | Apns消息体                          |
| ---------------------------------------- | ---------------------------------------- | -------------------------------- |
| 无                                        | identifer                                | HTTP/2的header中的 apns-collapse-id |
| 无                                        | content.attachments(UNNotificationAttachmen) | 可在UNNotificationService中修改       |
| applicationIconBadgeNumber（Integer）      | content.badge                            | badge                            |
| alertBody                                | content.body                             | alert的body                       |
| category                                 | content.categoryIdentifer                | category                         |
| alertLaunchImage                         | lanuchImageName（打开时启动图）                  | alert的lanch-image                |
| soundName（新无法转换为旧）                       | UNNotificationSound-soundNamed:          | sound                            |
| 无                                        | subtitle                                 | alert的subtitle                   |
| 无                                        | threadIdentifer（特殊标识符，自定义用）              | thread-id                        |
| alertTitle（iOS8.2以上）                     | title                                    | alert的title                      |
| userInfo                                 | userInfo                                 | 所有                               |
| fireDate                                 | UNNotificatonTimeInterval-trigger.timeInterval | 无                                |
| repeatInterval>0 或 repeatCalendar不为空或regionTriggersOnce为YES | trigger.repeat                           | 无                                |
| alertAction（alert模式下action名字）            | 无                                        | 无                                |
| region（触发位置）                             | UNLocationNotificationTrigger-trigger.region | 无                                |
| timeZone                                 | 无                                        | 无                                |
| repeatInterval（重复时间点）                    | UNNotificationTimeInterval/Calendar-nextTriggerDate转换 | 无                                |
| reatCalendar（重复的日历）                      | trigger的Class为UNNotificationPushTrigger  | 无                                |
| regionTriggersOnce                       | UNLocationNotificationTrigger-trigger.repeats | 无                                |
| hasAction（alert下是否有动作）                   | 无                                        | 无                                |

| UNNotificationAction                     | UIUserNotificationAction                 |
| ---------------------------------------- | ---------------------------------------- |
| identifer                                | Identifer                                |
| title                                    | title                                    |
| UNNotificationActionOptions-options      | activationMode+authenticationRequired+destructive共同决定 |
| UNTextInputNotificationAction/UNNotification（按钮种类） | UIUserNotificationActionBehavior-behavior（iOS9以上） |
| 无                                        | parameters                               |
| UNTextInputNotificationAction-textInputButtonTitle | 无                                        |
| UNTextInputNotificationAction-textInputPlaceholder | 无                                        |

| UNNotificationCategory              | UIUserNotificationCategory               |
| ----------------------------------- | ---------------------------------------- |
| identifer                           | Identifer                                |
| actions(UNNotificationAction)       | actionsForContext(UIUserNotificationAction) |
| intentIdentifers(支持Siri或苹果叫车框架的标识符) | 无                                        |
| options（点击后是否消失等等）                  | 无                                        |



iOS10的通知框架，和之前比简直焕然一新，整洁，优雅。还有一点需要注意的是，因为原来属于UIKit中，那么最好在主线程中操作。但由于老框架暂时还不会被废弃，所以在接入时，要注意新老框架的兼容和覆盖，避免出现不必要的Bug。

PS：此文又名《iOS10通知框架最全总结》

PPS：此文又名《iOS10通知框架，你看我就够了》

PPPS：此文又名《iOS10通知框架，你真的懂么？》

所有源码和[Demo](https://github.com/Nemocdz/iOS10NotificationTest)
如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

##### 参考链接

* [活久见的重构 - iOS 10 UserNotifications 框架解析](https://onevcat.com/2016/08/notification/)

* [AppleDocument-PaylodReference](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/PayloadKeyReference.html)

  ​

