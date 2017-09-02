---
title: "TableView和CollectionView通用的DataSource和Cell"
date: 2017-03-25T17:11:13+08:00
draft: false
---

刚进公司不到一个月，接到一个需求，把一个项目的界面改一下。看了项目里的视图，耦合性强，没有复用，改起UI来很费劲。新的界面是个列表视图，就寻思怎么写出一个后面接手的人改起UI不那么费劲的tableview，顺便把项目里老的collectionView也进行了类比，偷偷进行了重构。毕竟UI总是改来改去的，而常用的tableview和collecitonview更是应该让其扩展性更好，复用性更强。

<!--more-->

先从tableview讲起，tableview都很熟悉，一般就是由datasource，delegate，cell，cellitem这几部分组成。

### 打造一个通用性更强的DataSource

新建一个NSObject的子类，命名为CDZTableDataSource，并遵循<UITableViewDataSource>协议。对于datasource来说，必须实现的方法是``-(NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section``

和``-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath``

先看看第一个方法，返回每个section的row的数目，这个数字怎么来呢，就是section里的数据条数。如果不实现optional方法

``- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView``的话，默认只有一个section。所以我们首先要确定的就是section的数目，所以对应的，我们应该有section的model。新建一个NSObject的子类，命名为CDZSectionObject。那么这个section应该有些什么属性呢？对于一个section对象来说，应该持有一个row对应的model的数组，这样就可以确定每个section的row的数量了。可能还有一些sectionheader是数据等等。

```objective-c
@property (nonatomic, copy) NSString *headerTitle;
@property (nonatomic, copy) NSString *headerReuseIdentifer;
@property (nonatomic, strong) NSMutableArray *itemsArray;
```

回到datasouce，那么datasouce就可以加上这些属性。

```objective-c
@property (nonatomic, strong) NSMutableArray<CDZSectionObject *> *sectionsArray;
@property (nonatomic, strong) CDZSectionObject *firstSection;	
```

那么现在就可以实现第一个方法了，为了方便我们还可以默认有一个firstsection，方便只有一个section的情况使用。

```objective-c
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section{
    if (self.sectionsArray.count > section) {
        return self.sectionsArray[section].itemsArray.count;
    }
    return 0;
}


- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView{
    return self.sectionsArray ? self.sectionsArray.count : 0;
}

- (CDZSectionObject *)firstSection{
    if (!_firstSection) {
        _firstSection = [[CDZSectionObject alloc]init];
        [self.sectionsArray addObject:_firstSection];
    }
    return _firstSection;
}
```

轮到第二个方法了，配置cell的时候，我们需要知道cell对应的item而去配置cell，那么更抽象一点的话，也就是说datasource是建立了cellitem和cell之间的联系，也就是Model-View直接的联系。View本身做的工作也就是根据Model配置View的样子。所以这时候我们我们可以定义一个接口。

```objective-c
@protocol CDZCustomView
@required
- (void)setItem:(id)item;
@end
```

然后让我们的Cell遵守这个协议，也就是所有通用的Cell，需要在方法内实现item的解析。而其实对于cell也是一种view，这个接口可以让一些复用的view都遵守，比如headerview等等。对于cell我们还可以拓展这个协议。

```objective-c
@protocol CDZTableCell <CDZCustomView>
@required
+ (CGFloat)tableView:(UITableView *)tableView rowHeightForItem:(id)item;
@end
```

这样，cell的高度也可以由cell内部自己决定，适合各种各样的cell，当然也可以让autolayout决定。但是有一些老项目里的cell里面没有autolayout，所以返回高度通用性可能更强些。

回到datasource，我们解决了cell的model问题，下一个是，cell的class怎么决定呢？其实对于datasouce来说，cell的类型，往往是由对应的model决定的，也就是model的类型，决定可以使用的cell的种类。那么我们在datasouce里，只要给出一个接口，让使用者建立这种联系，item class - cell class的联系，并在内部用一个类似哈希表的东西储存起来，我们便可以根据item找到cell的class。简单的话用一个NSMutableDictionary就可以了。

```objective-c
- (void)setCellClass:(Class)cellClass forItemClass:(Class)itemClass{
    if (!itemClass || !cellClass) {
        return;
    }
    [self.cellDic setObject:cellClass forKey:NSStringFromClass(itemClass)];
}
```

而这样以后要更换view的时候，只要新建一个cell并重新set一下对应的item的就可以了，换model也类似。

那么现在就可以根据index找到对应的section和对应的row所对应的item所对应的cellclass了。

```objective-c
- (id)tableView:(UITableView *)tableView itemForRowAtIndexPath:(NSIndexPath *)indexPath{
    if (self.sectionsArray.count > indexPath.section) {
        if (self.sectionsArray[indexPath.section].itemsArray.count > indexPath.row) {
            return self.sectionsArray[indexPath.section].itemsArray[indexPath.row];
        }
    }
    return nil;
}

- (Class)tableView:(UITableView *)tableView cellClassForRowAtIndexPath:(NSIndexPath *)indexPath{
    id item = [self tableView:tableView itemForRowAtIndexPath:indexPath];
    Class cellClass = [self.cellDic objectForKey:NSStringFromClass([item class])];
    if (!cellClass) {
        cellClass = [UITableViewCell class];//没有对应关系的，取默认的UITalbeViewCell
    }
    return cellClass;
}

```

那么cellforrow方法就出来了

```objective-c
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    Class cellClass = [self tableView:tableView cellClassForRowAtIndexPath:indexPath];
    UITableViewCell<CDZTableCell> *cell = [tableView dequeueReusableCellWithIdentifier:NSStringFromClass(cellClass)];
    if (!cell) {
        cell = [[cellClass alloc]initWithStyle:UITableViewCellStyleDefault reuseIdentifier:NSStringFromClass(cellClass)];
    }
    if ([cell respondsToSelector:@selector(setItem:)]) {//检查是否有实现setItem的方法
        [cell setItem:[self tableView:tableView itemForRowAtIndexPath:indexPath]];
    }
    return cell;
}
```

到此DataSource就完成了。

### 如何使用呢？

让自定义的cell遵守<CDZTableCell>协议，内部实现setItem方法去配置view。一些老的cell也可以简单的遵守协议并把以前配置视图的方法写到setItem方法里就可以了。协议比继承耦合性更低些，即加即用。

让tableview的datasouce指定为一个CDZTabeDataSource，并设置item对应的cell类型就可以。更换cell和item只要重新建立关系即可，达到了datasource的复用和拓展性。对于各种tableview，就不用重复写datasource的代码了。例子如Demo

```objective-c
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{
    Class cellClass = [self.tableDataSource tableView:tableView cellClassForRowAtIndexPath:indexPath];
    return (cellClass == [UITableViewCell class]) ? 44.f : [cellClass tableView:tableView rowHeightForItem:[self.tableDataSource tableView:tableView itemForRowAtIndexPath:indexPath]];
}


- (UITableView *)tableView{
    if (!_tableView) {
        _tableView = [[UITableView alloc]initWithFrame:self.view.frame style:UITableViewStylePlain];
        _tableView.delegate = self;
        _tableView.dataSource = self.tableDataSource;
    }
    return _tableView;
}

- (CDZTableDataSource *)tableDataSource{
    if (!_tableDataSource) {
        _tableDataSource = [[CDZTableDataSource alloc]init];
        [_tableDataSource setCellClass:[CDZTableViewCell class] forItemClass:[CDZCellItem class]];
        _tableDataSource.firstSection.itemsArray = [[self mockItems] copy];
    }
    return _tableDataSource;
}
```

### CollecitonViewDatasource其实也类似

区别在于collecitonview的cell要注册，所以cellforrow方法有所不同

```objective-c
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    Class cellClass = [self collectionView:collectionView cellClassForRowAtIndexPath:indexPath];
    UICollectionViewCell<CDZCollectionCell> *cell = [collectionView dequeueReusableCellWithReuseIdentifier:NSStringFromClass(cellClass) forIndexPath:indexPath];
    if ([cell respondsToSelector:@selector(setItem:)]) {
        [cell setItem:[self collectionView:collectionView itemForRowAtIndexPath:indexPath]];
    }
    return cell;
}
```

其它基本类似，且collectionview和tableview还可以共用一个sectionobject。

## 最后

所有源码和[Demo](https://github.com/Nemocdz/CDZDataSourceDemo)
工作里，我们经常会因为赶需求而做一些复制粘贴。但也别忘了多思考如何复用，如何类比，虽然花掉现在一些时间，却会在日后的重构，修改，别人的修改上带来很多便利。

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

