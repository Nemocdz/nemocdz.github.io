---
title: "iOS中权限小Tips和监测相册变化"
date: 2016-12-14T23:05:13+08:00
draft: false
---

之前写了一个仿Snapseed的ImagePicker，见[Demo](https://github.com/Nemocdz/CDZImagePickerDemo)和之前的[文章](https://nemocdz.github.io/My-blog/post/ios%E4%B8%AD%E5%AE%9E%E7%8E%B0%E4%BB%BFsnapseed%E7%9A%84%E7%85%A7%E7%89%87%E9%80%89%E6%8B%A9%E5%99%A8/)。

之后发现有人说第一次未授权时collectionview会加载不出照片，发现没有在合适时候调用权限管理。iOS10的这个权限管理是info.plist的一个键值，没有回调方法。查询photoskit方法里，发现有一个权限回调方法，就想封装一个方法来友好的询问用户权限。

查询文档，相册权限PHAuthorizationStatus枚举类型有这么几种。

```objective-c
typedef NS_ENUM(NSInteger, PHAuthorizationStatus) {
    PHAuthorizationStatusNotDetermined = 0, // User has not yet made a choice with regards to this application 用户未决定
    PHAuthorizationStatusRestricted,        // This application is not authorized to access photo data.  一般是家长权限之类的拒绝
                                            // The user cannot change this application’s status, possibly due to active restrictions
                                            //   such as parental controls being in place.
    PHAuthorizationStatusDenied,            // User has explicitly denied this application access to photos data. 用户拒绝 
    PHAuthorizationStatusAuthorized         // User has authorized this application to access photos data. 用户允许
} PHOTOS_AVAILABLE_IOS_TVOS(8_0, 10_0);
```

其中，我们不希望用户去拒绝，达到`PHAuthorizationStatusDenied`这个状态，这样的话用户再要获取权限就要到设置里的隐私里找到我们的应用再手动打开，而没办法在app内去请求权限更改。那么我们希望暂时拒绝后的状态应该是`PHAuthorizationStatusNotDetermined `的，思路就是提示一个alertviewcontroller去询问，用户取消则不去调用真正的询问权限（相当于忽略），以便下次提醒在询问。用户同意则调用系统询问权限，这时一般用户也会再允许了。流程示意图如下。![屏幕快照 2016-12-14 22.15.02](http://ww1.sinaimg.cn/large/006tNc79gw1faqpzkg83ej31kw0uv769.jpg)

代码很简单，直接贴上来

```objective-c
- (void)showPermissionAlertInController{
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil message:@"需要你的图库的权限" preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction *action) {
        //do something
    }];
    UIAlertAction *requestAction = [UIAlertAction actionWithTitle:@"同意" style:UIAlertActionStyleDefault handler:^(UIAlertAction *action) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [PHPhotoLibrary requestAuthorization:^(PHAuthorizationStatus status) {
                if (status == PHAuthorizationStatusAuthorized) {
                    NSLog(@"用户同意授权相册");
                }else {
                    NSLog(@"用户拒绝授权相册");
                }
                dispatch_async(dispatch_get_main_queue(), ^{
                    //do something
                });
            }];

        });
    }];
    [alert addAction:cancelAction];
    [alert addAction:requestAction];
    [self presentViewController:alert animated:YES completion:nil];
}
```

再在需要用到权限的地方加上判断就好

``if ([PHPhotoLibrary authorizationStatus] == PHAuthorizationStatusAuthorized)``

---

后来发现如果调用选择器时在外部改变了照片（增加，删除）后collectionview不会改变，就研究了一下photoskit，发现有个``- (void)photoLibraryDidChange:(PHChange *)changeInfo ``检测相册变动，但之前需要注册为观察者并实现协议

`` PHPhotoLibraryChangeObserver`` 

```objective-c
- (void)viewDidLoad{
    [super viewDidLoad];
    [[PHPhotoLibrary sharedPhotoLibrary]registerChangeObserver:self];
   //other code
}
```



```objective-c
- (void)dealloc{
    [[PHPhotoLibrary sharedPhotoLibrary]unregisterChangeObserver:self];
}
```

然后实现方法，更新ui的话要在主线程里

```objective-c
- (void)photoLibraryDidChange:(PHChange *)changeInfo {
    dispatch_async(dispatch_get_main_queue(), ^{
        PHFetchResultChangeDetails *changes = [changeInfo changeDetailsForFetchResult:self.imageAssetsResult];
        if (changes) {
            self.photosDataSource.itemArray = [self getImageAssets];
            [self.photosView reloadData];
        }
    });
}
```

这个方法默认所有相册的变更都会通知，但是有些变动是我们不需要的，比如用户新建了一个新空相册，删除了一个旧相册等，这时我们可以用PHChange里的changeDetails去区分，比如``changeDetailsForFetchResult:(PHFetchResult *)object;``就是传入一组FetchResult，如果这组Result有变化，比如这组Result是Asset的集合，Asset增加减少，那么就会返回一个ChangeDetails，可以用此区分，但是如果是Asset内部的变化，就不会返回一个ChangeDetails。同样的如果这组Result是Collection的集合，那么Collection内部的变化，比如Collection里的Asset对象增加减少，就不会有返回。

具体可以见苹果官方的[例子](https://developer.apple.com/reference/photos/phphotolibrarychangeobserver?language=objc)

---

## 最后

这篇文章篇幅不多，作为一些补充，希望能帮到有需要的人。










