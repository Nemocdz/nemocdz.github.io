---
title: "Tableview支持不同种类的cell"
date: 2017-01-08T03:52:13+08:00
draft: false
---

在项目中，偶尔需要让tableview里支持不同种类的cell，比如微博的原创微博和别人转发的微博，就是两种cell。又或是类似支付宝的的timeline也有各种类型的cell。在同一个tableview里实现不同种类的cell，一般有两种方法，一种是把所有种类的cell先注册了，再根据不同的identifer去加载cell，一种是在init时创建不同的identifer的cell。效果图如下。

![](https://ws2.sinaimg.cn/large/006tKfTcly1fi1aan9mt6j30l812i0ud.jpg)

### 准备工作

创建一个基类的``CDZBaseCell``，基类cell拥有一些共用的属性和方法，如持有model，解析model。

创建不同的子类cell，以两个子类``CDZTypeACell`` ``CDZTypeBCell`` 为例，继承自``CDZBaseCell``，重写一些方法，如高度，显示视图等等。

Datasource中准备好判断index所在的cell种类的方法（如根据model的type属性等）

```objective-c
- (Class)cellClassAtIndexPath:(NSIndexPath *)indexPath{
    CDZTableviewItem *item = [self itemAtIndexPath:indexPath];
    switch (item.type) {
        case typeA:{
            return [CDZTypeACell class];
        }
            break;
        case typeB:{
            return [CDZTypeBCell class];
        }
            break;
    }
}

- (CDZTableviewItem *)itemAtIndexPath:(NSIndexPath *)indexPath{
    return self.itemsArray[indexPath.row];
}

- (NSString *)cellIdentiferAtIndexPath:(NSIndexPath *)indexPath{
    return NSStringFromClass([self cellClassAtIndexPath:indexPath]);
}
```

### 方法一：先注册，根据identifer去加载不同的cell

先在tableview创建时注册需要的不同种类，再判断index对应的种类，再根据identifer加载子类cell。

```objective-c
[self.tableview registerClass:[CDZTypeACell class] forCellReuseIdentifier:NSStringFromClass([CDZTypeBCell class])];
[self.tableView registerClass:[CDZTypeBCell class] forCellReuseIdentifier:NSStringFromClass([CDZTypeBCell class])];
```

并在``- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath``中根据重用标识加载cell。

```objective-c
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    CDZBaseCell *cell = [tableView dequeueReusableCellWithIdentifier:[self cellIdentiferAtIndexPath:indexPath] forIndexPath:indexPath];
    cell.item = [self itemAtIndexPath:indexPath];
    return cell;
}
```

### 方法二:在init时创建不同identifer的cell

在``- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath``中判断cell是否为nil，并根据index所在cell的种类初始化cell和其identifer。

```objective-c
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    CDZBaseCell *cell = [tableView dequeueReusableCellWithIdentifier:[self cellIdentiferAtIndexPath:indexPath]];
    if (!cell) {
        Class cls = [self cellClassAtIndexPath:indexPath];
        cell = [[cls alloc]initWithStyle:UITableViewCellStyleDefault reuseIdentifier:[self cellIdentiferAtIndexPath:indexPath]];
    }
    cell.item = [self itemAtIndexPath:indexPath];
    return cell;
}
```

### 总结

个人更喜欢第二种，苹果官方文档也推荐第二种方法去重用cell。我觉得优点是一个是在tableview划分MVC架构时，tableview创建时不需要知道cell的类型，而只需要知道datasouce，而datasource才是需要去分配cell类型的。第二个是tableviewcell的初始化方法并非只能用initWithStyle（collectionview必须先注册的原因则在于初始化方法只有initWithFrame）。而使用了注册，则是在复用池空时默认调用initWithStyle的方法，如果需要用别的方法初始化就不可以了。第一种方法可以用在有一些库需要先注册后才能调用的，比如自动计算cell高度的库``FDTemplateLayoutCell``。

### 最后

所有源码和[Demo](https://github.com/Nemocdz/CDZDifferenceCellDemo)

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

