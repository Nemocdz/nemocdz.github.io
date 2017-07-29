---
title: "TableView内嵌套CollectionView动态高度的实现（表单或图片分享）"
date: 2017-01-24T01:24:13+08:00
draft: false
---

在有社交分享平台属性的app中，我们经常看见类似有tableview中多图展示。不管是发布的表单界面中，还是社交动态的时间线的界面中，都需要根据图片数量动态变化界面。最近刚好写了一个这样的界面，花了点时间写了个Demo总结一下，希望可以帮助有需要的人。实现Demo效果如下图。

![](https://ws1.sinaimg.cn/large/006tKfTcly1fi1a8609dcg30ks1127t7.gif)

### 实现原理分析

初步思路是在tableview的cell中内嵌一个collecitonview，collectionview高度动态变化并是collectionview所在的tableview的cell的高度动态变化。总结起来我们需要这几件事：

1.实现一个tableview并自定义一种tableviewcell并实现高度自适应

2.在tableviewcell中实现collectionview并实现高度动态变化

3.自定义collectionviewcell中实现按钮点击事件（如删除，跳转），数据展示操作

### 1.实现tableview和自定义tableviewcell

tableview就很简单，storyboard或者代码写一下都可以。实现数据源协议啥的，很普通。要实现tableviewcell的高度自适应，一般来说有两种方式，一种是用iOS7后支持的cell的estimatedRowHeight和iOS8后支持的self-sizing cells（两者差不多，iOS8更完善一些），另一种是用孙源大神的第三方开源库，可以看这篇[文章](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)，两者共同之处都是需要设置cell里contenview的元素对cell的contenview的四个边的布局约束，换言之要让cell里的元素把cell四边“撑”起来。Demo里使用了原生的self-sizing cells来高度自适应，需要下面两句代码。

```objective-c
self.tableview.estimatedRowHeight = 100.f;//数字为大致估算高度，比如有些50有些100可以估算75左右
self.tableview.rowHeight = UITableViewAutomaticDimension;
```

如果tableview是用代码创建的，那么rowheight属性的默认参数就是``UITableViewAutomaticDimension``，不需要第二行代码。而在storyboard或xib里拖的默认rowheight是拖的storyboard属性菜单里的，需要更改为``UITableViewAutomaticDimension``。

接着自定义一个``CDZTableViewCell``，~~并用xib拖上一个collectionview并设置对contenview的autolayout约束。用Masonry之类的第三方布局库或原生进行代码约束也可以。然后在tableview里用``registerNib``方法注册一下自定义的cell。~~

**因为TableView执行reloadData方法时，会把所有cell从VisableCells池中移除，并从同一复用标识的复用池中取出Cell加入视图中，这个时候cell的高度已经确定好，但indexPath的row顺序有可能不是原来的，所以不能复用。也就是虽然每一个Cell功能一致，但是由于高度和里面图片数量不一样，并不能互相复用。在Storyboard中可以用静态cell分别放入collectionView实现。这里为了不重复放collectionView用了代码书写TableView。**

```objective-c
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    NSString *identifer = [NSString stringWithFormat:@"CDZTableViewCell%ld",indexPath.row];//唯一复用标识，相当于不复用
    CDZTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:identifer];
    if (!cell) {
        cell  = [CDZTableViewCell.alloc initWithStyle:UITableViewCellStyleDefault reuseIdentifier:identifer];
    }
    cell.delegate = self;
    return cell;
}
```

当然要实现动态高度变化，我们还需要让cell在合适的时候通知tableview应该更新数据和布局了，即调用tableview自身的``reloadData``方法。这里我用delegate实现，通知的话也可以，但是通知的要明确接受者和发送者的对应关系，不然有些情况会接受对象不明确（比如实现了两个类似的tableview在视图里）。在cell的头文件里创建delegate。

```objective-c
@protocol CDZTableViewCellDelegate<NSObject>
- (void)didChangeCell:(UITableViewCell *)cell;
@end
@interface CDZTableViewCell : UITableViewCell
@property (nonatomic,assign) id<CDZTableViewCellDelegate> delegate;
@end
```

然后让tableview遵守``CDZTableViewCellDelegate``并在tableview的datasource或delegate方法里设置cell的delegate，比如

```objective-c
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    //......
    cell.delegate = self;
    //......
}
```

也可以在delegate里的``-(void)collectionView:(UICollectionView *)collectionView willDisplayCell:(UICollectionViewCell *)cell forItemAtIndexPath:(NSIndexPath *)indexPath``里设置（感觉这个方法更好些，datasource应该处理数据相关的，而且有时候会复用抽离出来）。

### 2.实现collectionview和自定义collectionviewcell

collectionview在约束上，除了实现collecitionview对tableviewcell的上下左右约束外，还要实现其高度约束，因为本质collectionview是scrollview的子类，实现scollview的autolayout其实就是需要实现对其撑起来contentview的约束，而动态实现contentview的高度，则需要我们更新contenview的高度布局约束的值。collectionview也是类似的，我们先设定一个collectionview的高度约束，再在数据更新时更新高度约束。

这里再提一句，collectionview本身和tableview类似，系统本身也有self-sizing cells的api，如下：

```objective-c
self.collectionViewFlowLayout.estimatedItemSize = CGSizeMake(125, 100);
self.collectionViewFlowLayout.itemSize = UICollectionViewFlowLayoutAutomaticSize;
```

同样也要实现collectionviewcell里元素对cell四边的约束，“撑”起来自动计算高度，但是不知道为何用了之后在一个cell的时候会莫名其妙居中，且cell的indexPath有些错乱，根据cell的indexPath找出来的cell是错误的，不知道是啥问题，若有人知道可以告诉一声囧。

这里就不用collectionview的大小自适应了，常规的用设置itemsize的大小就好了。在tableviewcell``awakeFromNib``方法中，我们让collectionview``reloadData``更新一下高度布局约束为collectionview的真实高度并调用刚才自定义的delegate去让tableview``reloadData``从而让tableview的cell高度自适应。

```objective-c
- (void)reloadCell{
    [self.collectionView reloadData];
    [self.collectionView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.height.equalTo(@(self.collectionView.collectionViewLayout.collectionViewContentSize.height));
    }];
    [self.delegate didChangeCell:self];
}
```

collectionview的真实大小其实是他的layout对象的collectionviewcontentsize的值。所以我们需要在每次数据发生改变时，更新一下高度约束的值。

还有就是实现collectionview的delegate，点击后执行操作，在赋值在cell对应的model里等操作（比如调用相册等）。注意当点击最后一个collectioncell时，要在它的后面插入一个数据，也就是说，最后一个总是会保持在最后。

```objective-c
- (void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath{
    //载入数据，如图片等
    CDZCollectionViewItem *item = [CDZCollectionViewItem new];
    item.image = [UIImage imageNamed:@"example"];
    if ((indexPath.row == self.itemsArray.count - 1)) {
        [self.itemsArray insertObject:item atIndex:self.itemsArray.count - 1];
    }
    else{
        self.itemsArray[indexPath.row] = item;
    }
    [self reloadCell];
}
```

### 3.自定义collectionviewcell并实现删除按钮

先自定义一个``CDZCollectionViewCell``，并用xib拖上一个imageview用于展示图片，一个button用于点击关闭cell。并定义好解析item的方法。item中设定一个delBtnHidden属性用于设定cell里的button是否隐藏。

```objective-c
- (void)setItem:(CDZCollectionViewItem *)item{
    //  解析需要的数据
    self.imageView.image = item.image;
    self.delButton.hidden = item.delBtnHidden;
}
```

并定义一个delegate用于将按钮点击事件回传给collectionview并删除数据。

```objective-c
@protocol CDZCollectionCellDelegate<NSObject>
- (void)didDelete:(UICollectionViewCell *)cell;
@end
  
@interface CDZCollectionViewCell : UICollectionViewCell
@property (strong, nonatomic) CDZCollectionViewItem *item;
@property (assign, nonatomic) id<CDZCollectionCellDelegate> delegate;
@end
```

并在点击按钮的事件中调用delegate，把cell本身传回tableview，从而找到cell对应的item。

```objective-c
- (IBAction)delCell:(UIButton *)sender{
    if ([self.delegate respondsToSelector:@selector(didDelete:)]){
        [self.delegate didDelete:self];
    }
}
```

然后使collectionview所在的tableviewcell遵守``CDZCollectionCellDelegate``。并和之前一样，在collectionview的datasource或delegate中设置delegate。

```objective-c
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    //......
    cell.delegate = self;
    //......
}
```

并调用cell的delegate的方法，通过cell去找到对应的indexPath

```objective-c
- (void)didDelete:(UICollectionViewCell *)cell{
    NSIndexPath *indexPath = [self.collectionView indexPathForCell:cell];
    [self.itemsArray removeObjectAtIndex:indexPath.row];
    [self reloadCell];
}
```

第一个cell是没有关闭按钮的，那么只要把第一个item的delBthHidden属性设为YES就可以了。

### 最后

所有源码和[Demo](https://github.com/Nemocdz/CDZCollectionInTableDemo)

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

