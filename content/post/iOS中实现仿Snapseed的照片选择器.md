---
title: "iOS中实现仿Snapseed的照片选择器"
date: 2016-12-04T01:55:13+08:00
draft: false
---

ImagePickerController（图片选择器）是iOS开发中一个很常用的UI控件。作为摄影爱好者，而Google出品的摄影后期App，Snapseed中那个图片选择器，好看又实用，便尝试着仿写了一个。集成后一行代码即可实现下图效果。
![](http://ww2.sinaimg.cn/large/006y8mN6gw1fae4wlsm89g30ku1121gx.gif)
实现原理从界面说起。
界面可以分解为一个背景图层用于点击关闭（空白处点击消失），一个横向可滚动的collectionview用于显示图库里的图片，一个tableview用于存放功能按钮。

##1. 功能按钮的tableview
先从简单的tableview说起，从小模块到大模块应该是以下方面。

* 自定义数组数据对象item方便调整按钮的文字图片数量功能
* 自定义tableview的cell方便样式调整，cell中属性根据对应item设置
* 自定义tableview的datasourece在之中设置cell和item的对应关系
* tableview的deleagte，设置点击事件
* 点击功能实现，设置UIImagePickerController
* tableview的view初始化，设置datasource，view绘制

自定义item对象，item对象定义了action的显示名字，类型，图片

```objective-c
@property(nonatomic, copy) NSString *actionTitle;
@property(nonatomic, strong) UIImage *actionImage;
@property(nonatomic, assign) CDZImagePickerActionType actionType;
```
其中CDZImagePickerActionType可用来定义按钮动作类型，可以用枚举来定义

```objective-c
typedef NS_ENUM(NSInteger, CDZImagePickerActionType) {
    CDZImagePickerCameraAction,
    CDZImagePickerLibraryAction,
    CDZImagePickerRecentAction,
    CDZImagePickerCloseAction
};
```
定义一个init方法用来快捷初始化item对象

```objective-c
- (instancetype)initWithTitle:(NSString *)titele withActionType:(CDZImagePickerActionType)type withImage:(UIImage *)image{
    self = [super init];
    if (self) {
        _actionTitle = titele;
        _actionType = type;
        _actionImage = image;
    }
    return self;
}
```
---
自定义cell对象，封装一个方法来对应item和cell的关系

```objective-c
- (void)setCellFromItem:(CDZImagePickerActionsItem *)item{
    self.textLabel.text = item.actionTitle;
    self.imageView.image = item.actionImage;
}
```
可以重写layoutSubviews方法达到cell中文字，图片样式自定义的效果，比如可以写有图片和无图片的样式布局

```objective-c
- (void)layoutSubviews{
    [super layoutSubviews];
    self.selectionStyle = UITableViewCellSelectionStyleNone;//点击不变色
    self.imageView.frame = CGRectMake(20, 15, 22, 22);
    self.imageView.contentMode = UIViewContentModeScaleAspectFit;
    self.textLabel.font = [UIFont systemFontOfSize:16.0f];
    self.textLabel.textColor = [UIColor colorWithRed:0.30 green:0.30 blue:0.30 alpha:1.00];
    if (self.imageView.image){
        self.textLabel.frame = CGRectMake(60, 2, 200, 48);
        self.textLabel.textAlignment = NSTextAlignmentLeft;
    }
    else{
        self.textLabel.textAlignment = NSTextAlignmentCenter;
    }
}
```
---
自定义datasource，这一步可以根据个人选择，个人习惯是把datasource从viewcontroller里抽离，可以让controller更少代码,datasource有数组对象用于存放item

``@property (nonatomic, strong) NSMutableArray *itemArray;``
实现datasource方法

```objective-c
#pragma mark - tableViewDataSourceRequried
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section{
    return self.itemArray.count;
}
		

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    CDZImagePickerActionsItem *item = self.itemArray[indexPath.row];
    CDZImagePickerActionsCell *cell = [tableView dequeueReusableCellWithIdentifier:NSStringFromClass([CDZImagePickerActionsCell class])];
    if (!cell) {
        cell = [[CDZImagePickerActionsCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:NSStringFromClass([CDZImagePickerActionsCell class])];
    }
    [cell setCellFromItem:item];//将cell和item对应起来
    return cell;
}
```
---
实现delegate和点击事件

```objective-c
#pragma mark - tableViewDelegate

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{
    return actionsViewCellHeight;
}


- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath{
    CDZImagePickerActionsItem *item = self.actionArray[indexPath.row];
    [self doActionsWithType:item.actionType];
}
```

```objective-c
#pragma mark - actions
- (void)doActionsWithType:(CDZImagePickerActionType)type{
    switch (type) {
        case  CDZImagePickerCameraAction:
            [self openCamera];
            break;
        case   CDZImagePickerRecentAction:
            [self openRecentImage];
            break;
        case CDZImagePickerLibraryAction:
            [self openLibrary];
            break;
        case CDZImagePickerCloseAction:
            [self closeAction];
            break;
    }   
}

- (void)openCamera{
    if ([UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]){
        UIImagePickerController *pickerController = [[UIImagePickerController alloc]init];
        pickerController.delegate = self;
        pickerController.sourceType = UIImagePickerControllerSourceTypeCamera;
        [self presentViewController:pickerController animated:NO completion:nil];
        NSLog(@"打开相机");
    }
}

- (void)openLibrary{
    UIImagePickerController *pickerController = [[UIImagePickerController alloc]init];
    pickerController.delegate = self;
    pickerController.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
    [self presentViewController:pickerController animated:NO completion:nil];
    NSLog(@"打开图库");
    
}

- (void)closeAction{
    [self dismissViewControllerAnimated:YES completion:nil];
    NSLog(@"关闭按钮");
}


- (void)openRecentImage{
    [[PHImageManager defaultManager]requestImageForAsset:self.photosDataSource.itemArray[0] targetSize:PHImageManagerMaximumSize contentMode:PHImageContentModeAspectFill options:nil resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
        self.resultImage = result;
        [self dismissViewControllerAnimated:YES completion:nil];
        NSLog(@"打开最新图片");
    }];
}
```
实现imagepicker的delegate（**记得实现UINavigationControllerDelegate，不可只实现UIImagePickerControllerDelegate，会报错**）

```objective-c
#pragma mark - imagePickerController delegate

- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(nonnull NSDictionary<NSString *,id> *)info{
    UIImage *image = info[UIImagePickerControllerOriginalImage];
    if(picker.sourceType == UIImagePickerControllerSourceTypeCamera){
        UIImageWriteToSavedPhotosAlbum(image, self, @selector(image:didFinishSavingWithError:contextInfo:), nil);
    }
    self.resultImage = image;
    [picker dismissViewControllerAnimated:NO completion:nil];
    [self dismissViewControllerAnimated:YES completion:nil];
    NSLog(@"从相机或图库获取图片");
}
```
iOS10还要检查一下相机和图库的权限，记得在info.plist中添加两行，不然会崩溃
![](http://ww3.sinaimg.cn/large/006y8mN6gw1fadyvsos06j30r4024754.jpg)
---
最后在是tableview初始化与绘制，在datasource初始化自己需要的按钮item数组（acctionArray）

```objective-c
- (CDZImagePickerActionsDataSource *)actionsDataSource{
    if (!_actionsDataSource) {
        _actionsDataSource = [[CDZImagePickerActionsDataSource alloc]init];
        _actionsDataSource.itemArray = self.actionArray;
    }
    return _actionsDataSource;
}
```
```objective-c
- (UITableView *)actionView{
    if (!_actionView) {
        CGFloat actionsViewHeight = actionsViewCellHeight * self.actionArray.count;
        _actionView = [[UITableView alloc]initWithFrame:CGRectMake(0,SCREEN_HEIGHT - actionsViewHeight ,SCREEN_WIDTH, actionsViewHeight) style:UITableViewStylePlain];
        _actionView.scrollEnabled = NO; //不需要滑动
        _actionView.separatorStyle = UITableViewCellSeparatorStyleNone; //分割线去除
        _actionView.delegate = self;
        _actionView.dataSource = self.actionsDataSource;
    }
    return _actionView;
}
```
---

---
##2.展示照片的collecionview
collecionview的实现思路和tableview类似

* 自定义collectionview的cell实现样式调整
* 自定义Datasource解析并设置当前的cell对应的图片
* 用Photokit的方法抓取图库的照片
* 实现collectionviewflowlayout的delegate实时调整cell的大小，边距
* 实现collecitonview的delegate，设置点击事件
* 实现collectionview的初始化和视图绘制

cell的自定义主要是重写其initWithFrame方法添加imageview（**注意collectionviewcell只用initWithFrame方式初始化，tableviewcell初始化方法有多种**），并在layoutSubviews里改变imageview的大小,和tableview类似写一个setCellFromItem的方法解析item数据。而Photokit获取的图片对象为phasset，用PHImageMange的方法根据将其解析为size为cell的大小uimage，加快加载速度，按需加载（关于Photokit的优点不再赘述，iOS8以后苹果只保留了Photokit用于获取系统图片）

photokit获取时，注意两个地方，targetSize传入的是pixel，而不是平时用的size，且option里的resizeMode默认为None（不缩放），不重写resizeMode属性那么只会抓取不缩放的图，在意质量可以设置resizeMode为Exact（精确），追求速度可以设置为Fast(这样不会完全按照设置的targetSize去获取图片)

```objective-c
- (instancetype)initWithFrame:(CGRect)frame{
    self = [super initWithFrame:frame];
    if (self) {
        [self.contentView addSubview:self.photoImageView];
    }
    return self;
}

- (void)layoutSubviews{
    [super layoutSubviews];
    self.photoImageView.frame = self.contentView.bounds;
}

- (void)setCellFromItem:(PHAsset *)asset{
  PHImageRequestOptions *options = [[PHImageRequestOptions alloc]init];
    options.resizeMode = PHImageRequestOptionsResizeModeFast;
    [[PHImageManager defaultManager]requestImageForAsset:asset targetSize:[UIScreen mainScreen].bounds.size contentMode:PHImageContentModeAspectFill options:options resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
        self.photoImageView.image = result;
    }];
}

- (UIImageView *)photoImageView{
    if (!_photoImageView) {
        _photoImageView = [[UIImageView alloc]init];
        _photoImageView.contentMode = UIViewContentModeScaleAspectFill;
    }
    return _photoImageView;
}
```

---
datasource也类似，定义一个itemArray用于存放图片对象phasset
``@property (nonatomic, strong) NSMutableArray *itemArray;``
实现datasource方法，和tableview类似（**不用判断cell是nil是因为collectionviewcell的初始化只有一种**）

```objective-c
#pragma mark - collectionViewDataSourceRequried

- (NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section{
    return self.itemArray.count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    PHAsset *item = self.itemArray[indexPath.row];
    CDZImagePickerPhotosCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:NSStringFromClass([CDZImagePickerPhotosCell class]) forIndexPath:indexPath];
    [cell setCellFromItem:item];
    return cell;
}
```
---
在controller中获取全部照片
再用photokit的方法把所有图片获取到数组里

```objective-c
- (NSMutableArray *)getImageAssets{
    self.imageAssetsResult = [PHAsset fetchAssetsWithMediaType:PHAssetMediaTypeImage options:nil];
    NSMutableArray *assets = [NSMutableArray new];
    for (PHAsset *asset in self.imageAssetsResult){
        [assets insertObject:asset atIndex:0];
    }
    return assets;
}
```
---
collectionview的布局，cell的大小排列和间距等，都是由collectionviewlayout决定的，布局是线性的话，推荐用官方的子类collectionviewflowlayout，其deleagate是collectionviewdeleagateflowlayout，是collectionviewdelegate的子delegate。

```objective-c
- (UICollectionViewFlowLayout *)photosFlowLayout{
    if (!_photosFlowLayout) {
        _photosFlowLayout = [UICollectionViewFlowLayout new];
        _photosFlowLayout.scrollDirection = UICollectionViewScrollDirectionHorizontal; //水平滚动
    }
    return _photosFlowLayout;
}
```
```objective-c
#pragma mark - collectionViewDelegateFlowLayout
//根据asset的长宽设定每个cell的size
- (CGSize)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout *)collectionViewLayout sizeForItemAtIndexPath:(NSIndexPath *)indexPath{
    PHAsset *asset = self.photosDataSource.itemArray[indexPath.row];
    CGFloat height = photosViewHeight - 2 * photosViewInset;
    CGFloat aspectRatio = asset.pixelWidth / (CGFloat)asset.pixelHeight;
    CGFloat width = height * aspectRatio;
    CGSize size = CGSizeMake(width, height);
    return size;
}

//设置整个collectionview上下左右间距
- (UIEdgeInsets) collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout *)collectionViewLayout insetForSectionAtIndex:(NSInteger)section{
    return UIEdgeInsetsMake(photosViewInset, photosViewInSet, photosViewInset, photosViewInset);
}

//设置cell之间的间距
- (CGFloat) collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout *)collectionViewLayout minimumInteritemSpacingForSectionAtIndex:(NSInteger)section{
    return photosViewInset;
}
```
---
设置collecionview的delegate和点击事件

```objective-c
#pragma mark - collectionViewDelegate

- (void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath{
    [[PHImageManager defaultManager]requestImageForAsset:self.photosArray[indexPath.row] targetSize:PHImageManagerMaximumSize contentMode:PHImageContentModeAspectFill options:nil resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
        self.resultImage = result;
        [self dismissViewControllerAnimated:YES completion:nil];
        NSLog(@"已选择图片");
    }];
}
```
---
最后就是collectionview的初始化和绘制和tableview类似，多了一步注册collectionview的cell（**collectionviewcell必须让collecionview注册重用标识，而tableviewcell可以在自己init方法里注册**）

```objective-c
- (UICollectionView *)photosView{
    if (!_photosView){
        CGFloat actionsViewHeight = actionsViewCellHeight * self.actionArray.count;
        _photosView = [[UICollectionView alloc]initWithFrame:CGRectMake(0, SCREEN_HEIGHT - actionsViewHeight - photosViewHeight , SCREEN_WIDTH, photosViewHeight) collectionViewLayout:self.photosFlowLayout];
        _photosView.delegate = self;
        _photosView.dataSource = self.photosDataSource;
        _photosView.backgroundColor = [UIColor whiteColor];
        _photosView.showsHorizontalScrollIndicator = NO;
        [_photosView registerClass:[CDZImagePickerPhotosCell class] forCellWithReuseIdentifier:NSStringFromClass([CDZImagePickerPhotosCell class])];
    }
    return _photosView;
}
```
```objective-c
- (CDZImagePickerPhotosDataSource *)photosDataSource{
    if (!_photosDataSource){
        _photosDataSource = [[CDZImagePickerPhotosDataSource alloc]init];
        _photosDataSource.itemArray = [self getImageAssets];
    }
    return _photosDataSource;
}
```
---
##3.透明图层（用于实现空白处点击消失）
比较简单，增加一个tap手势即可，直接上代码（记得实现UIGestureRecognizerDelegate）

```objective-c
- (UIView *)backgroundView{
    if (!_backgroundView){
        CGFloat actionsViewHeight = actionsViewCellHeight * self.actionArray.count;
        _backgroundView =[[UIView alloc]initWithFrame:CGRectMake(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT - photosViewHeight - actionsViewHeight)];
        _backgroundView.backgroundColor.opaque = YES;//设置为透明
        _backgroundView.userInteractionEnabled = YES;
        UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc]initWithTarget:self action:@selector(dissPicker:)];
        [_backgroundView addGestureRecognizer:tap];
    }
    return _backgroundView;
}
```

##4.收尾&封装
定义一个block用于回调传照片
``typedef void (^CDZImageResultBlock) (UIImage *image);``
一个内部block属性
``@property (nonatomic ,copy) CDZImageResultBlock block;``
封装方法

```objective-c
- (void)openPickerInController:(UIViewController *)controller withImageBlock:(CDZImageResultBlock)imageBlock{
    self.modalPresentationStyle = UIModalPresentationOverCurrentContext;//iOS8上默认presentviewcontroller不透明，需设置style
    self.block = imageBlock;
    [controller presentViewController:self animated:YES completion:nil];
}
```
在Dealloc方法里调用block回调

```objective-c
- (void)dealloc{
    self.block(self.resultImage);
    NSLog(@"ImagePicker已销毁");
}
```

##5.使用方法

将CDZImagePicker文件夹拖入项目

```objective-c
CDZImagePickerViewController *imagePickerController = [[CDZImagePickerViewController alloc]init];
[imagePickerController openPickerInController:self withImageBlock:^(UIImage *image) {
        if (image) { //如果没选照片会回调nil，若想让之前照片不变就加上判断
           //yourcode
        }
       // yourcode
}];
```

也可以自定义CDZActionsItem自定义文字和图片

```objective-c
imagePickerController.actionArray = [NSMutableArray arrayWithObjects:
                      [[CDZImagePickerActionsItem alloc]initWithTitle:@"打开设备上的图片" withActionType:CDZImagePickerLibraryAction withImage:[UIImage imageNamed:@"phone-icon.png"]],
                      [[CDZImagePickerActionsItem alloc]initWithTitle:@"相机" withActionType:CDZImagePickerCameraAction withImage:[UIImage imageNamed:@"camera-icon.png"]],
                      [[CDZImagePickerActionsItem alloc]initWithTitle:@"打开最新图片" withActionType:CDZImagePickerRecentAction withImage:[UIImage imageNamed:@"clock-icon.png"]],
                      nil];
```
效果就和snapseed很像啦！
![](http://ww4.sinaimg.cn/large/006y8mN6gw1fae4wt2mk0g30ku1124oj.gif)



## 最后

所有源码和[Demo](https://github.com/Nemocdz/CDZImagePickerDemo)
虽然不是什么很难的轮子，但希望能分享给有需要的人。

关于后续的处理照片变更和获取相册权限另写了一篇[文章](https://nemocdz.github.io/My-blog/post/ios%E4%B8%AD%E6%9D%83%E9%99%90%E5%B0%81%E8%A3%85%E5%B0%8Ftips%E5%92%8C%E7%9B%91%E6%B5%8B%E7%9B%B8%E5%86%8C%E5%8F%98%E5%8C%96/)，欢迎阅读。

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流





























