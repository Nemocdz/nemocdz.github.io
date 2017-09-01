---
title: "iOS原生二维码界面开发的一些注意点"
date: 2017-05-14T01:33:13+08:00
draft: false
---

之前的[文章](https://nemocdz.github.io/My-blog/post/ios%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%94%9F%E7%9A%84%E4%BA%8C%E7%BB%B4%E7%A0%81%E6%89%AB%E6%8F%8F%E7%95%8C%E9%9D%A2/)是在ViewController里实现了原生二维码扫描的功能，后来觉得VC太重了，便把功能抽成了一个View，在抽象的过程中也仔细捉摸了一些没有仔细思考过的问题，特此记录下来。

![IMG_1130](http://ww4.sinaimg.cn/large/006tNbRwly1ffk93o9dj4j30ku112afv.jpg)

#### 1.扫描区域的扫描线动画

像微信等很多二维码扫描界面，总有一个上下移动的扫描线动画。一开始用我用NSTimer配合view的frame改变实现，后来发现不仅容易导致NSTimer的循环引用，外界调用也容易开启多个定时器，已经在某些时候动画会卡顿。便换成了用CABasicAnimation实现，通过改变view的y值，并把设置repeatCount为最大，达到扫描线上下动的效果。

```objective-c
- (void)addScanLineAnimation{
    self.scanLine.hidden = NO;
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"transform.translation.y"];
    animation.fromValue = @(0);
    animation.toValue = @(self.scanRect.size.height - scanLineWidth);
    animation.duration = scanTime;
    animation.repeatCount = OPEN_MAX;
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    [self.scanLine.layer addAnimation:animation forKey:scanLineAnimationName];
}


- (void)removeScanLineAnimation{
    [self.scanLine.layer removeAnimationForKey:scanLineAnimationName];
    self.scanLine.hidden = YES;
}
```

取消时把动画移除即可。

### 2.扫描线的渐变色拖尾

这个用CAGradientLayer实现即可，设置渐变起始点和终止点，传入颜色数组和对应的起始位置（0~1），即可。

```objective-c
- (UIView *)scanLine{
    if (!_scanLine) {
        _scanLine = [[UIView alloc]initWithFrame:CGRectMake(self.scanRect.origin.x,self.scanRect.origin.y, self.scanRect.size.width, scanLineWidth)];
        _scanLine.hidden = YES;
        CAGradientLayer *gradient = [CAGradientLayer layer];
        gradient.startPoint = CGPointMake(0.5, 0);
        gradient.endPoint = CGPointMake(0.5, 1);
        gradient.frame = _scanLine.layer.bounds;
        gradient.colors = @[(__bridge id)[self.scanLineColor colorWithAlphaComponent:0].CGColor,(__bridge id)[self.scanLineColor colorWithAlphaComponent:0.4f].CGColor,(__bridge id)self.scanLineColor.CGColor];
        gradient.locations = @[@0,@0.96,@0.97];
        [_scanLine.layer addSublayer:gradient];
    }
    return _scanLine;
}
```

### 3.设置AVFoundation原生二维码类的一些容易崩溃的地方

具体x相关类别在上篇[文章](http://www.jianshu.com/p/ad7827a8a0e6)已经有所介绍，讲讲一些容易崩溃的点，崩溃主要集中在设置AVCaptureMetadataOutput的metadaObjectTypes属性时，所以在设置这个数组时，最好先检查数组里的值是否在其availableMetaDataObjectTypes属性里。像这样

```objective-c
if (![self.dataOutput.availableMetadataObjectTypes containsObject:AVMetadataObjectTypeQRCode]) {
     NSLog(@"The camera unsupport for QRCode.");
}
self.dataOutput.metadataObjectTypes = @[AVMetadataObjectTypeQRCode];
```

而一般的崩溃是因为availableMetaDataObjectTypes为空，而造成空的原因是AVCaptureSession没有addInput和addOutput造成。其中input有一个需要注意的地方，在setSessionPreset，也就是取样的视频分辨率的时候，如果设置了一个摄像头不支持的分辨率，那么input是canAddInput会返回NO。并不是所有设备都支持1080p视频分辨率的，所以我们在setSessionPreset时，可以这样

```objective-c
if ([self.device supportsAVCaptureSessionPreset:AVCaptureSessionPreset1920x1080]) {
    [self.session setSessionPreset:AVCaptureSessionPreset1920x1080];
}
else{
    [self.session setSessionPreset:AVCaptureSessionPresetHigh];
}    
```

AVCaptureSessionPresetHigh就是该设备所支持最高的分辨率。而单纯通过设备ScreenHigh来区分是否支持1080p视频分辨率并不可靠，4s以后是支持1080p的，但是还有一些touch设备，iPad设备，已经某些后摄像头坏掉的手机（会自动调用前摄像头，比如我的5sTAT）上，就会出现分辨率不支持，canAddInput返回NO，没有input，availavleMetadataObjectTypes为空，如果不注意就会产生崩溃。

### 4.关于扫描区域的正确计算方法

之前提到扫描区域的interestedRect是一个值是0~1的CGRect，后来我实验发现，这个0~1相对的并不是屏幕的Frame，而是AVCaptureVideoPreviewLayer的Frame值，而我们习惯把AVCaptureVideoPreviewLayer的frame值等于view的frame值，从而才在计算时用view的frame值去算百分比。但后来我发现正确的转换方法是用AVCaptureVideoPreviewLayer里的metadataOutputRectOfInterestForRect进行正常Rect->扫描区域坐标系Rect转换，另外AVCaptureVideoPreviewLayer还提供了rectForMetadataOutputRectOfInterest进行扫描区域Rect->正常坐标系Rect的转换，以及CGPoint的相关转换方法。而且这个转换需要放在session的StartRunning方法之后才会生效。

```objective-c
[self.session startRunning];
self.dataOutput.rectOfInterest = [self.previewLayer metadataOutputRectOfInterestForRect:self.scanRect];
```

### 5.耗时操作放在异步并发线程中处理

AVFoundation里设置相机相关的方法比较耗时，可以放在异步并发线程中处理。

### 最后

个人封装了一个CDZQRScanView简单易用。支持Cocoapods也可以直接把两个文件拉进项目里。

所有源码和[Demo](https://github.com/Nemocdz/CDZQRScanView)

使用时可以把VC两个文件拖进项目里就可。

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流
