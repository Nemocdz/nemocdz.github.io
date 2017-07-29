---
title: "iOS中实现静态Cell通用的TableViewManager"
date: 2017-05-14T21:36:13+08:00
draft: false
---

在App中，比如侧边栏，设置，个人界面，编辑表单，总会有一些地方用到一些静态Cell构成的TableView，这些界面的特点是，Cell种类繁多，Model种类繁多，点击事件处理分散但单调（页面跳转，编辑textFiled），重复写tableViewDelegate和datasource的相关方法，但不会用到一些奇怪的datasource和delegate方法，而且显示内容一般是固定的，每个页面Cell的数量不会很多。像笔者所在的项目里的侧边栏模块，里面有很多tableView，由于没有使用StoryBoard且很久之前写的，充斥着各种if...else if用model的某个key值进行判断的逻辑去让cell去显示和隐藏某些视图，已经进行点击跳转逻辑，更有甚者，有``if (indexPath == 0)``之类的通过行数判断去处理的逻辑。增加新的cell时，便要在各种地方再加上else if的新判断，而做新界面时，也没有很好的方法进行复用。笔者结合之前[文章](http://www.jianshu.com/p/1f7304634600)进行进一步思考，抽象出一个适合这种特点tableView的管理类，记录一下思路供参考。

![](https://ws4.sinaimg.cn/large/006tKfTcly1fi1a1xvl8qj30p00r7ach.jpg)

### 关于TableViewDatasource的思考

在之前的[文章（iOS通用的DataSource）](http://www.jianshu.com/p/1f7304634600)中，已经对Datasource和SectionObject进行了抽象，datasource本质是告诉Cell如何去处理Model的桥梁，之前笔者是通过维护一个字典去一一对应Model的种类和Cell种类的，但后来发现，有时候Model和Cell的种类并不一定是1对1的，比如有些时候cell只要只要展示一个title，可能使用者只想写一个String去对应，而不用去建一个Model，在静态不变的Model中，这种情况更突出，因为并不需要从网络解析数据，不用用反射机制去进行json转model的处理，有时候，一个NSDictionary甚至一个NSString或者Bool就可以满足一个cell的展示需求，那么对于这种可能是多Cell对多Model（String也可以是一种Model）的情况，且每个TableView的Cell数量不多（一般设置页不会超过20个），可以考虑为在字典里每个model配置对应的Cell类型的键值对。而且因为Model种类不一，我们也需要手动Add到SectionObject里，那么我们可以把这个字典分在每个sectionObject里去管理。在section类里添加以下方法，而model的唯一性可以用hash属性标识，默认hash属性就是对象的内存地址。

```objective-c
- (void)addItem:(id<NSObject>)item cellClass:(Class)cellClass{
    if (!item) {
        return;
    }
    [_items addObject:item];
    if (!cellClass) {
        return;
    }
    NSString *itemHash = [NSString stringWithFormat:@"%tu",item.hash];
    [self.sectionCellDic setObject:cellClass forKey:itemHash];
}
```

然后再增加外部通过indexPath读取对应Cell Class的方法

```objective-c
- (Class)cellClassIndexPath:(NSIndexPath *)indexPath{
    return [self cellClassForItem:[self itemForIndexPath:indexPath]];
}

- (Class)cellClassForItem:(id<NSObject>)item{
    NSString *itemHash = [NSString stringWithFormat:@"%tu",item.hash];
    Class cellClass = [self.sectionCellDic objectForKey:itemHash];
    return cellClass;
}

- (id)itemForIndexPath:(NSIndexPath *)indexPath{
    if (self.items.count > indexPath.row) {
        return self.items[indexPath.row];
    }
    return nil;
}
```

### 接管TableViewDelegate

和很多动态Cell会有一些奇奇怪怪的Delegate不同，静态cell主要就是展示和点击事件，一般会用到以下原生delegate

```objective-c
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath;
- (void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath;
- (UIView *)tableView:(UITableView *)tableView viewForFooterInSection:(NSInteger)sectionIndex;
- (CGFloat)tableView:(UITableView *)tableView heightForFooterInSection:(NSInteger)sectionIndex;
- (UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)sectionIndex;
- (CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)sectionIndex;
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;
```

所以我们可以让TableViewManager接管这些协议的实现，同时新建一个TableViewManagerDelegate为UITableViewDelegate的子协议，并开放这些协议给使用Manager的人，也就是可以让使用者覆写这些delegate。

比如heightForRowAtIndexPath方法，就可以在判断外部是否实现，否则则内部实现。

```objective-c
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{
    if ([self.delegate respondsToSelector:@selector(tableView:heightForRowAtIndexPath:)]) {
        return [self.delegate tableView:tableView heightForRowAtIndexPath:indexPath];
    }
    Class cellClass = [self tableView:tableView cellClassForRowAtIndexPath:indexPath];
    id item = [self tableView:tableView itemForRowAtIndexPath:indexPath];
    return (cellClass == [UITableViewCell class])?44.f:[cellClass tableView:tableView rowHeightForItem:item];
}
```

同理，这些协议的默认实现可以根据实际情况作调整，比如对于didSelectRowAtIndexPath，我们也可以带上其item再返回给外部，带上item的好处，等会会讲到。

```objective-c
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    if ([self.delegate respondsToSelector:@selector(tableView:didSelectItem:atIndexPath:)]) {
        id item = [self tableView:tableView itemForRowAtIndexPath:indexPath];
        [self.delegate tableView:tableView didSelectItem:item atIndexPath:indexPath];
        [tableView deselectRowAtIndexPath:indexPath animated:YES];
    }
    else if ([self.delegate respondsToSelector:@selector(tableView:didSelectRowAtIndexPath:)]) {
        [self.delegate tableView:tableView didSelectRowAtIndexPath:indexPath];
    }
}
```

### Cell的协议里有啥

之前文章讲过，我们可以通过一个CellProtocol去指定configItem的方法让cell自己去解析Model，同时我们也可以让这个协议丰富一点，比如加上CellDelegate的回调，可以让Cell把信息传到外部，这在一些Cell里面的点击事件，或者开关（UISwitch），表单编辑时都很有用，所以我们可以加上delegate

```objective-c
@protocol CDZTableViewCellDelegate <NSObject>
@optional
- (void)didTriggleCell:(UITableViewCell*)cell message:(id)message;
@end

@protocol CDZTableViewCell<NSObject>
@required
- (void)configWithItem:(id<NSObject>)item;
+ (CGFloat)tableView:(UITableView *)tableView rowHeightForItem:(id)item;
@optional
- (void)setCDZCellDelegate:(id<CDZTableViewCellDelegate>)objectDelegate;
@end
```

而这个Message的执行者应该是外部调用者而不是Manager本身，所以对于Cell和外部来说，它虽然是个id但实际类型是外部和Cell内部统一知道的，比如是个Bool值表示开关状态，一个NSDictionary里面包含信息，或者是一个NSString。而Manager只要做一个中转就好了。这里我在Manager里把Cell转换成了对应的Item及IndexPath传给外部，因为Cell是可复用的，实际关心的应该是Model。

```objective-c
- (void)didTriggleCell:(UITableViewCell *)cell message:(id)message{
    if ([self.delegate respondsToSelector:@selector(receiveCellMessage:atIndexPath:item:)]) {
        NSIndexPath *indexPath = [self.tableView indexPathForCell:cell];
        id item = [self tableView:self.tableView itemForRowAtIndexPath:indexPath];
        [self.delegate receiveCellMessage:message atIndexPath:indexPath item:item];
    }
}
```

### 初始化方法

```objective-c
- (instancetype)initWithTableView:(UITableView *)tableView delegate:(id<CDZTableViewManagerDelegate>)objectDelegate{
    self = [super init];
    if (self){
        tableView.delegate = self;
        tableView.dataSource = self;
        tableView.separatorStyle = UITableViewCellSeparatorStyleNone;
        _sections = [NSMutableArray array];
        _delegate = objectDelegate;
        self.tableView = tableView;
    }
    return self;
}
```

最后外界只需要初始化一个manager，即可不用写Datasource和Delegate了，因为都被Manager接管了。

### item的一些小技巧

对应一些初展示类或者开关操作的item，我们可以用简单的Bool，NSString，NSDictionry作为Model，而有点击事件的Model，我们可以在model类里增加一个tapBlock，这样我们可以在配置item时顺便配置其点击事件。因为静态cell中的点击事件往往各不相同，所以配置在tapBlock可以让代码更加统一。

```objective-c
TestItem *itemD = [[TestItem alloc]init];
itemD.title = @"itemD";
itemD.tapBlock = ^(TestItem *item) {
   NSLog(@"%@ tap",item.title);
};
[firstSection addItem:itemD cellClass:[TestBCell class]];
```

这样不仅好找，一次性把相关的操作配置完成，而且在执行点击事件时，不用再关心具体的点击事件，进行item的识别分类判断。

```objective-c
- (void)tableView:(UITableView *)tableView didSelectItem:(id)item atIndexPath:(NSIndexPath *)indexPath{
    if ([item isMemberOfClass:[TestItem class]]) {
        TestItem *cellItem = (TestItem *)item;
        if (cellItem.tapBlock) {
            cellItem.tapBlock(cellItem);
        }
    }
}
```

### 最后

通过这些的Manager，就可以随心所欲搭配model和cell，并把配置写在一起，方便修改，查找，同时也不用重复写Delegate和Datasource，而对老的Cell和Model也有很好的兼容。底下是一个混搭的例子。

```objective-c
- (NSMutableArray <CDZTableViewSection *>*)sections{
    NSMutableArray <CDZTableViewSection *> *sections = [NSMutableArray array];
    CDZTableViewSection *firstSection = [[CDZTableViewSection alloc]init];
  
    NSString *itemA = @"itemA";
    [firstSection addItem:itemA cellClass:[TestACell class]];
    
  	NSDictionary *itemB = @{@"title" : @"itemB"};
    [firstSection addItem:itemB cellClass:[TestACell class]];
   
    TestItem *itemC = [[TestItem alloc]init];
    itemC.title = @"itemC";
    itemC.tapBlock = ^(TestItem *item) {
        NSLog(@"%@ tap",item.title);
    };
    [firstSection addItem:itemC cellClass:[TestBCell class]];
  
    TestItem *itemD = [[TestItem alloc]init];
    itemD.title = @"itemD";
    itemD.tapBlock = ^(TestItem *item) {
        NSLog(@"%@ tap",item.title);
    };
    [firstSection addItem:itemD cellClass:[TestBCell class]];
  
    [sections addObject:firstSection];
    return sections;
}
```

所有源码和[Demo](https://github.com/Nemocdz/CDZTableViewManager)
可以直接拿文件去使用或修改，也支持Cocoapods.

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

