---
title: "iOS中实现多列联动的PickerView"
date: 2017-03-29T01:33:13+08:00
draft: false
---

去年年底写了一个自定义的PickerView，[文章](http://www.jianshu.com/p/bf7f304ee308)在这里。当时对Block和各种复用思想还理解很浅薄，只是单纯的实现了一个单列数据的Picker，前几天还看到评论有人给我提了些建议，便花了一天时间做了些升级，做了多列联动的实现。多列联动的Picker很常见，比如用在地址选择，如省-市-区中。网上搜索的例子要不是只写死指定的列数，要不然就是指定数据格式严格而复杂。我的强迫症又犯了，要写，就写一个，简单而且，能无限列联动的。样子如下：

![](https://ww4.sinaimg.cn/large/006tNc79ly1fe330ca1zjg30ca0m8wil.gif)

### 类比而来的实现思路

首先对于数据联动，也就是，前面一列的所选择数据决定后面一列拥有的所有数据。这个模型马上让我联想到了生活中一个例子-文件管理器。列之间数据之间的关系，其实就像一颗文件目录结构树。后面列的数据是前面数据的叶子节点。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fi1a6r35ztj30m60e6wgr.jpg)

而对于文件的目录结构的获取，由于之前在公司开发过日志模块的相关组件，还残存着记忆。主要思路就是遍历+递归。那么构建数据结构时，就很清晰了。新建一个NSObject的子类CDZPickerComponetObject，那么这个Model需要有些啥呢，就如文件夹一般，需要名字，用于显示在Picker上。还需要一个可以存放自身的数组，存放自己“文件夹“”里的“文件夹”。

```objective-c
@property (nonatomic, strong) NSMutableArray<CDZPickerComponentObject *> *subArray;
@property (nonatomic, copy) NSString *text;

- (instancetype)initWithText:(NSString *)text subArray:(NSMutableArray *)array;
- (instancetype)initWithText:(NSString *)text;
```

而对于外界，只要传入一个``NSArray<CDZPickerComponentObject *> *componetArray;``就可以里，这里面的object，想怎么嵌套怎么嵌套，比如

```objective-c
   CDZPickerComponentObject *guangdong = [[CDZPickerComponentObject alloc]initWithText:@"广东省"];
    CDZPickerComponentObject *guangzhou = [[CDZPickerComponentObject alloc]initWithText:@"广州市"];
    CDZPickerComponentObject *chaozhou = [[CDZPickerComponentObject alloc]initWithText:@"潮州市"];
    guangdong.subArray = [NSMutableArray arrayWithArray:@[guangzhou, chaozhou]];
    CDZPickerComponentObject *haizhu = [[CDZPickerComponentObject alloc]initWithText:@"海珠区"];
    guangzhou.subArray = [NSMutableArray arrayWithObject:haizhu];
```

### 实现Picker的几个关键方法

在上篇[文章](http://www.jianshu.com/p/bf7f304ee308)里也提过，UIPickerView主要是实现其Delegate和DataSource的几个方法。

首先实现最简单的一个``- (NSInteger)numberOfComponentsInPickerView:(UIPickerView *)pickerView``

这个方法返回的是列数，列数是什么，其实就是“文件夹”嵌套的层级，而我们只要对CDZPickerComponetObject取自身的任意一个数据，再重复操作并进行计数，取到没有subArray为止，便可以得到列数了。

先实现一个取自身数据的方法如下：

```objective-c
- (CDZPickerComponentObject *)objectAtIndex:(NSInteger)index inObject:(CDZPickerComponentObject *)object{
    if (object.subArray.count > index) {
        return object.subArray[index];
    }
    return nil;
}
```

循环操作即可，得到index即为列数，也就是树的深度。

```objective-c
CDZPickerComponentObject *object = self.componetArray.firstObject;
for (_numberOfComponents = 1;; _numberOfComponents++) {
	object = [self objectAtIndex:0 inObject:object];
    if (!object) {
        break;
       }
}
```

接下来是``- (NSInteger)pickerView:(UIPickerView *)pickerView numberOfRowsInComponent:(NSInteger)component``先从第一行列考虑，第一列的行数，其实就是componetArray的个数，而其它列的行数呢，依赖于当前上一列选择的行数，所以我们需要先考虑所选择的行数。也就是，前面列的行数的组合，决定了当前列的数据个数。所以重要的是，前面列的组合，有了组合，当前列的情况就被确定下来了。而前面的组合那么多种，怎么确定呢？``- (NSInteger)selectedRowInComponent:(NSInteger)component; ``这个Api可以根据列数得到列数选择的行数，知道列数的组合后，我们就可以一层层找到每列的数组的个数，也就是行数。那每次获取行数都要一层层遍历么？我们可以把每列的数组，存放起来，也就是一个二维的数组，行数和列数，而每次改变某一列时，只要改变当列对应的数组，这样就不用每次遍历了。新增一个二维数组rowsArray。

``@property (nonatomic, strong) NSMutableArray<NSMutableArray <CDZPickerComponentObject *> *> *rowsArray;``

而这个二维数组会在什么时候改变呢？当然是``- (void)pickerView:(UIPickerView *)pickerView didSelectRow:(NSInteger)row inComponent:(NSInteger)component``中了，改变这一列的行数，其实改变的是下一列的数组，通过这一列的行数对应的object对应的subArray，就是下一列的数组。还要注意，当我们选中第一列的数据时，第二列应该根据第一列的数据变化，也就是，当这一列选中时，我们应该调用下一列并默认选中他的第一行，先调用选择第一行，再调用第一行的回调didSelectRow并重新加载那一列的数据。方法里的代码如下：

```objective-c
 if (component < (_numberOfComponents - 1)) {
 	 NSMutableArray<CDZPickerComponentObject *> *tmp = self.rowsArray[component];
     if (tmp.count > 0) {
        tmp = tmp[row].subArray;
     }
     [self.rowsArray setObject:((tmp.count > 0) ? tmp : [NSMutableArray array])  atIndexedSubscript:component + 1];
     [self pickerView:pickerView didSelectRow:0 inComponent:component + 1];
     [pickerView selectRow:0 inComponent:component + 1 animated:NO];
 }
 [pickerView reloadComponent:component];
```

这个二维数组的初始值也就是都是第一行对应的数组们，可以在之前的循环里顺便生成。

```objective-c
_rowsArray = [NSMutableArray array];
CDZPickerComponentObject *object = self.componetArray.firstObject;
[_rowsArray setObject:[NSMutableArray arrayWithArray:self.componetArray] atIndexedSubscript:0];
for (_numberOfComponents = 1;; _numberOfComponents++) {
    [_rowsArray setObject:object.subArray atIndexedSubscript:_numberOfComponents];
     object = [self objectAtIndex:0 inObject:object];
     if (!object) {
         break;
      }
 }
```

剩下就是	``- (UIView *)pickerView:(UIPickerView *)pickerView viewForRow:(NSInteger)row forComponent:(NSInteger)component reusingView:(UIView *)view``了，也就是找到二维数组对应的object。

很简单

```objective-c
NSArray<CDZPickerComponentObject *> *tmp = self.rowsArray[component];
if (tmp.count > 0) {
    genderLabel.text = tmp[row].text;
}
```

剩下的就是，在block里回调数据了，把每行选中的值，返回一个字符串数组就好了，这样方便行之间的数据分开。

```objective-c
for (NSInteger index = 0; index < _numberOfComponents; index++) {
	NSInteger indexRow = [self.pickerView selectedRowInComponent:index];
	NSMutableArray<CDZPickerComponentObject *> *tmp = self.rowsArray[index];
	if (tmp.count > 0) {
		[resultArray addObject:tmp[indexRow].text];
	}
}
```

### 多行不联动怎么办

那就更简单了，传入一个二维数组。新建一个属性``@property (nonatomic, strong) NSArray<NSArray <NSString*> *> *stringArrays;``然后其它方法，

``- (NSInteger)numberOfComponentsInPickerView:(UIPickerView *)pickerView``的返回值是字符串数组的个数，``- (NSInteger)pickerView:(UIPickerView *)pickerView numberOfRowsInComponent:(NSInteger)component``是数组里第componet个数组的个数。``- (UIView *)pickerView:(UIPickerView *)pickerView viewForRow:(NSInteger)row forComponent:(NSInteger)component reusingView:(UIView *)view``里很简单

```objective-c
NSArray<NSString *> *tmp = self.stringArrays[component];
if (tmp.count > 0) {
	genderLabel.text = tmp[row];
}
```

回调的stringArray也很简单

```objective-c
for (NSInteger index = 0; index < _numberOfComponents; index++) {
	NSInteger indexRow = [self.pickerView selectedRowInComponent:index];
	NSArray<NSString *> *tmp = self.stringArrays[index];
	if (tmp.count > 0) {
		[resultArray addObject:tmp[indexRow]];
	}
}
```

单行的也很简单，转换成一行的去思考就可以了。

### 如何使用呢？

在项目中导入CDZPicker，文件只有四个。或者Podfile加上pod 'CDZPicker' 然后cocoapods安装。导入import 'CDZPicker.h'

- 单列的:

传入一个字符串数组即可

```objective-c
[CDZPicker showPickerInView:self.view withStrings:@[@"objective-c",@"java",@"python",@"php"] confirm:^(NSArray<NSString *> *stringArray) {
    self.label.text = stringArray.firstObject;
}cancel:^{
    //your code
 }];
```

- 多列不联动：

传入多个字符串数组的数组即可

```objective-c
[CDZPicker showPickerInView:self.view withStringArrays:@[@[@"MacOS",@"Windows",@"Linux",@"Ubuntu"],@[@"Xcode",@"VSCode",@"Sublime",@"Atom"]] confirm:^(NSArray<NSString *> *stringArray) {
    self.label.text = [stringArray componentsJoinedByString:@"+"];
} cancel:^{
    // your code
 }];
```

- 多列联动：

先建立数据之间的嵌套关系

```objective-c
CDZPickerComponentObject *haizhu = [[CDZPickerComponentObject alloc]initWithText:@"海珠区"];
CDZPickerComponentObject *yuexiu = [[CDZPickerComponentObject alloc]initWithText:@"越秀区"];
    
CDZPickerComponentObject *guangzhou = [[CDZPickerComponentObject alloc]initWithText:@"广州市"];
guangzhou.subArray = [NSMutableArray arrayWithObjects:haizhu,yuexiu, nil];
    
CDZPickerComponentObject *xiangqiao = [[CDZPickerComponentObject alloc]initWithText:@"湘桥区"];
CDZPickerComponentObject *chaozhou = [[CDZPickerComponentObject alloc]initWithText:@"潮州市"];
chaozhou.subArray = [NSMutableArray arrayWithObjects:xiangqiao, nil];
    
CDZPickerComponentObject *guangdong = [[CDZPickerComponentObject alloc]initWithText:@"广东省"];
guangdong.subArray = [NSMutableArray arrayWithObjects:guangzhou,chaozhou, nil];
    
CDZPickerComponentObject *pixian = [[CDZPickerComponentObject alloc]initWithText:@"郫县"];

CDZPickerComponentObject *chengdu = [[CDZPickerComponentObject alloc]initWithText:@"成都市"];
chengdu.subArray = [NSMutableArray arrayWithObjects:pixian, nil];
    
CDZPickerComponentObject *leshan = [[CDZPickerComponentObject alloc]initWithText:@"乐山市"];
    
CDZPickerComponentObject *sichuan = [[CDZPickerComponentObject alloc]initWithText:@"四川省"];
sichuan.subArray = [NSMutableArray arrayWithObjects:chengdu,leshan, nil];
```

然后在传进去就可以了

```objective-c
[CDZPicker showPickerInView:self.view withComponents:@[guangdong,sichuan] confirm:^(NSArray<NSString *> *stringArray) {
	self.label.text = [stringArray componentsJoinedByString:@","];
}cancel:^{
	 //your code
 }];
```

## 最后

所有源码和[Demo](https://github.com/Nemocdz/CDZPicker)
如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流

#### 