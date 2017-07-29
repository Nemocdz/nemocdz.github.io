---
title: "iOS中实现一个支持小数的星星评分组件"
date: 2017-05-21T19:15:23+08:00
draft: false
---

在很多电商，外卖，餐饮型应用里，都会在商品结束后评价中有一个星星组件。核心思路就是用UIControl并自定义实现其中的trackTouch的几个方法。而显示不到一个的星星，比如半个星星的思路是根据分数切割星星的图像并显示其中一部分。实现后效果如下。

![2017-05-21 19_12_04](https://ws2.sinaimg.cn/large/006tNc79ly1fft73ku93og30da0n0tfa.gif)

### 单个星星的实现

对于单个星星的实现，先考虑星星有三个状态，完全置灰状态，完全高亮状态，根据百分比半高亮状态。而我这边用的是UIButton来实现，因为UIButton本身已经有普通，高亮，选择的状态。主要实现的就是百分比高亮状态。我们可以根据百分比将图像进行裁剪，让新图像的宽度只有百分比所占的整个图像的宽度。但是这时候调用setImage，会发现图片处于整个button中间。这是因为UIButton的imageView的contentMode默认是AspectFit的，而AspectFit是默认居中的。Contentmode中的UIViewContentModeLeft，是不会把长边缩放到imageView的边长的。况且，直接改变UIButton里的ImageView的ContentMode是没有效果的。要通过UIControl的contentVerticalAlignment属性（垂直）和contentHorizontalAlignment属性（水平）来达到类似ImageView的contentMode的效果。但是这两个属性都没有带Fit后缀的选项，也就是说并不会根据边长等比缩放。所以需要先把图片缩放到和Button大小一致。

![屏幕快照 2017-05-21 下午6.32.58](https://ws3.sinaimg.cn/large/006tNc79ly1fft5yrflcoj30dl06c3za.jpg)



之后，我们就可以按百分比裁剪图片

```objective-c
+ (UIImage *)croppedImage:(UIImage *)image fraction:(CGFloat)fractonPart{
    CGFloat width = image.size.width * fractonPart * image.scale;
    CGRect newFrame = CGRectMake(0, 0, width , image.size.height * image.scale);
    CGImageRef resultImage = CGImageCreateWithImageInRect(image.CGImage, newFrame);
    UIImage *result = [UIImage imageWithCGImage:resultImage scale:image.scale orientation:image.imageOrientation];
    CGImageRelease(resultImage);
    return result;
}
```

并在BackgroundImage设置为灰色的星星图像，设置为Button的highlighted状态

```objective-c
- (void)setFractionPart:(CGFloat)fractionPart{
    if (fractionPart == 0) {
        return;
    }
    UIImage *image = [CDZStarButton croppedImage:self.highlightedImage fraction:fractionPart];
    self.imageView.contentMode = UIViewContentModeScaleAspectFit;
    self.contentHorizontalAlignment = UIControlContentHorizontalAlignmentLeft;
    self.contentVerticalAlignment = UIControlContentVerticalAlignmentFill;
    [self setImage:image forState:UIControlStateHighlighted];
    [self setBackgroundImage:self.normalImage forState:UIControlStateHighlighted];
    self.selected = NO;
    self.highlighted = YES;
}
```

而全高亮的状态可以设置为UIButtonde点击态

```objective-c
- (void)setHighlightedImage:(UIImage *)highlightedImage{
    _highlightedImage = [CDZStarButton reSizeImage:highlightedImage toSize:self.frame.size];
    [self setImage:_highlightedImage forState:UIControlStateSelected];
}
```

而点击事件交由给上层的UIControl去处理，因为点击的时候，除了点击的星星本身，之前的星星也应该处于完全高亮状态。在上层初始化按钮的时候把其userInteractionEnabled属性设置为NO即可，这样触摸事件就会忽略传递到Button去做事件处理，从而从响应者链传递上去交由父视图的UIControl处理。

如果点击到星星的一半，我们应该把点击点转换成小数部分给上层。C语言的round函数可以四舍五入，这里的10代表保留一位小数，可以根据实际情况保留位数，一般评分很少需要更高精度的。

```objective-c
- (CGFloat)fractionPartOfPoint:(CGPoint)point{
    CGFloat fractionPart =  (point.x - self.frame.origin.x) / self.frame.size.width;
    return round(fractionPart * 10) / 10;
}
```

到这里一个可以显示分数的星星按钮就完成了，接下来就是在上层的UIControl去处理触摸事件的响应。

### UIControl部分的实现

主要分两块，星星按钮的布局，触摸事件响应。

#### 布局

首先根据星星的数量一个个添加上视图，用UIView的tag来表示对应第几个星星按钮。

```objective-c
- (void)setupView {
    for (NSInteger index = 0; index < self.numberOfStars; index++) {
        CDZStarButton *starButton = [CDZStarButton.alloc initWithSize:self.starSize];
        starButton.tag = index;
        starButton.normalImage = self.normalStarImage;
        starButton.highlightedImage = self.highlightedStarImage;
	    starButton.userInteractionEnabled = NO;//关闭事件响应，交由UIControl本身处理
        [self addSubview:starButton];
    }
}
```

然后是计算每个星星的位置，计算间隔和上下边距并layout

```objective-c
- (void)layoutSubviews {
    [super layoutSubviews];
    for (NSInteger index = 0; index < self.numberOfStars; index ++) {
        CDZStarButton *starButton =  [self starForTag:index];
        CGFloat newY = (self.frame.size.height - self.starSize.height) / 2;
        CGFloat margin = 0;
        if (self.numberOfStars > 1) {
            margin = (self.frame.size.width - self.starSize.width * self.numberOfStars) / (self.numberOfStars - 1);
        }
        starButton.frame = CGRectMake((self.starSize.width + margin) * index, newY, self.starSize.width, self.starSize.height);
    }
}
```

这时，我们可以加上两个方法去找到对应的星星button，根据tag和根据点击的CGPoint

```objective-c
- (CDZStarButton *)starForPoint:(CGPoint)point {
    for (NSInteger i = 0; i < self.numberOfStars; i++) {
        CDZStarButton *starButton = [self starForTag:i];
        if (CGRectContainsPoint(starButton.frame, point)) {
            return starButton;
        }
    }
    return nil;
}

- (CDZStarButton *)starForTag:(NSInteger)tag {
    __block UIView *target;
    [self.subviews enumerateObjectsUsingBlock:^(__kindof UIView * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        if (obj.tag == tag) {
            target = obj;
            *stop = YES;
        }
    }];
    return (CDZStarButton *)target;
}
```

#### 处理触摸事件

我们先加上两个方法。一个是从第一个到某个星星开始从左到右依次点亮，一个是从最后一个星星到某个星星从右到左依次熄灭。

```objective-c
- (void)starsDownToIndex:(NSInteger)index {
    for (NSInteger i = self.numberOfStars; i > index; --i) {
        CDZStarButton *starButton = [self starForTag:i];
        starButton.selected = NO;
        starButton.highlighted = NO;
    }
}

- (void)starsUpToIndex:(NSInteger)index {
    for (NSInteger i = 0; i <= index; i++) {
        CDZStarButton *starButton = [self starForTag:i];
        starButton.selected = YES;
        starButton.highlighted = NO;
    }
}
```

然后设置一个评分的属性，重写其setter方法。allowFraction，用来判断组件是否需要分数表示，floor函数是取最大整数，相当于直接去除小数点后面的数字。判断评分的整数部分是否已经亮着，亮着那么说明从左到右最后一个亮着的右边，反之在左边，分别调用从右到左依次熄灭，或从左到右依次点亮的方法，最后再设置分数部分。

```objective-c
- (void)setScore:(CGFloat)score{
    if (_score == score) {
        return;
    }
    _score = score;
    NSInteger index = floor(score);
    CGFloat fractionPart = score - index;;
    if (!self.isAllowFraction || fractionPart == 0) {
        index -= 1;
    }
    CDZStarButton *starButton = [self starForTag:index];
    if (starButton.selected) {
        [self starsDownToIndex:index];
    }
    else{
        [self starsUpToIndex:index];
    }
    starButton.fractionPart = fractionPart;
}
```

主要用到UIControl的四个方法

![屏幕快照 2017-05-21 下午6.31.56](https://ws4.sinaimg.cn/large/006tNc79ly1fft5y5gq92j30vt02475g.jpg)

第一个是开始点击的时候的事件处理，第二个是手指未抬起在屏幕上继续移动的事件处理，第三个是离开屏幕，第四个是因为别的特殊情况事件被结束的处理。

点击时候只要确定点击的星星，确定其小数部分，然后调用评分属性的setter方法就好了。

```objective-c
- (BOOL)beginTrackingWithTouch:(UITouch *)touch withEvent:(UIEvent *)event {
    CGPoint point = [touch locationInView:self];
    CDZStarButton *pressedStar = [self starForPoint:point];
    if (pressedStar) {
        self.currentStar = pressedStar;
        NSInteger index = pressedStar.tag;
        CGFloat fractionPart = 1;
        if (self.isAllowFraction) {
            fractionPart = [pressedStar fractionPartOfPoint:point];
        }
        self.score = index + fractionPart;
    }
    return YES;
}
```

移动处理除了和点击一样的判断逻辑，还要注意手指移开了星星之外的地方，分为所在星星的左边（当前星星熄灭），右边（当前星星完全点亮）两种。

```objective-c
- (BOOL)continueTrackingWithTouch:(UITouch *)touch withEvent:(UIEvent *)event {
    CGPoint point = [touch locationInView:self];
    CDZStarButton *pressedStar = [self starForPoint:point];
    if (pressedStar) {
        self.currentStar = pressedStar;
        NSInteger index = pressedStar.tag;
        CGFloat fractionPart = 1;
        if (self.isAllowFraction) {
            fractionPart = [pressedStar fractionPartOfPoint:point];
        }
        self.score = index + fractionPart;
    }
    else{
      	//移到了当前星星的左边
        if (point.x < self.currentStar.frame.origin.x) {
            self.score = self.currentStar.tag;
        }
      	//移到了当前星星的右边
        else if (point.x > (self.currentStar.frame.origin.x + self.currentStar.frame.size.width)){
            self.score = self.currentStar.tag + 1;
        }
    }
    return YES;
}
```

而完成触摸操作时，我们就可以用一个回调将当前的评分传给外界。

```objective-c
@protocol CDZStarsControlDelegate<NSObject>
@optional

/**
 回调星星改变后的分数

 @param starsControl 星星组件
 @param score 分数
 */
- (void)starsControl:(CDZStarsControl *)starsControl didChangeScore:(CGFloat)score;
@end
```

```objective-c
- (void)endTrackingWithTouch:(UITouch *)touch withEvent:(UIEvent *)event {
    [super endTrackingWithTouch:touch withEvent:event];
    if ([self.delegate respondsToSelector:@selector(starsControl:didChangeScore:)]) {
        [self.delegate starsControl:self didChangeScore:self.score];
    }
}


- (void)cancelTrackingWithEvent:(UIEvent *)event {
    [super cancelTrackingWithEvent:event];
    if ([self.delegate respondsToSelector:@selector(starsControl:didChangeScore:)]) {
        [self.delegate starsControl:self didChangeScore:self.score];
    }
}
```

#### 封装

```objective-c
- (instancetype)initWithFrame:(CGRect)frame
                        stars:(NSInteger)number
                     starSize:(CGSize)size
              noramlStarImage:(UIImage *)normalImage
         highlightedStarImage:(UIImage *)highlightedImage{
    if (self = [super initWithFrame:frame]) {
        _numberOfStars = number;
        _normalStarImage = normalImage;
        _highlightedStarImage = highlightedImage;
        _starSize = size;
        _allowFraction = NO;
        self.clipsToBounds = YES;
        self.backgroundColor = [UIColor clearColor];
        [self setupView];
    }
    return self;
}
```

开放score属性和allowFraction属性即可。使用起来也很简单。

```objective-c
 CDZStarsControl *starsControl = [CDZStarsControl.alloc initWithFrame:CGRectMake(0, 100, self.view.frame.size.width, 50) stars:5 starSize:CGSizeMake(50, 50) noramlStarImage:[UIImage imageNamed:@"star_normal"] highlightedStarImage:[UIImage imageNamed:@"star_highlighted"]];
    starsControl.delegate = self;
    starsControl.allowFraction = YES;
    starsControl.score = 2.6f;
    [self.view addSubview:starsControl];
```

然后在delegate里处理分数改变后的操作即可。

### 最后

所有源码和[Demo](https://github.com/Nemocdz/CDZStarsControl)
可以直接拿文件去使用或修改，也支持Cocoapods.

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流
有任何问题欢迎评论私信或者提issue