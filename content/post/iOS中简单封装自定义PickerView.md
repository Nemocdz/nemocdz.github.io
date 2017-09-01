---
title: "iOS中简单封装自定义PickerView"
date: 2016-11-19T02:28:13+08:00
draft: false
---

UIPickerView是iOS开发中，相当常用的一个UI控件，用于滚动选择选项。也是项目中经常复用的一个控件，封装成一个统一风格的库，可以减少很多代码量。一般还会在PickerView上加上Toolbar和确定取消按钮。
点击button弹出picker，并改变指定label的值，效果如下图所示。
![](http://ww1.sinaimg.cn/large/006tNc79gw1f9wp3v4z00g30ku112agf.gif)
最终目的是在viewcontroller中button的event reponse中调用封装好的库的方法进行传值及操作。

```objectivec
- (IBAction)showPicker:(UIButton *)sender {
    NSArray *array = @[@"电子科技大学",@"清华大学",@"四川大学",@"华中科技大学",@"西安电子科技大学"];
    [CDZPicker showPickerInView:self.view withObjectsArray:array withlastString:self.label.text withStringBlock:^(NSString *string) {
        self.label.text = string;
    }];
    }
```

实现思路从数据，视图，按钮处理三个方面说
###1. 数据

* 传入数据（上层view对象，数组array，最后的值）
* 改变的值（利用block传值回调）
* UIPickerView的Delegate和DataSource方法的实现

上层view对象和数组array没啥可说的，关于取消按钮的实现，个人想法是把弹出picker改变前最后的值传进view的内部并存储起来，等待按钮的动作响应判断是否取出。（但个人感觉把这个值传进去实现不太优雅，但水平不够没想到其它实现方法，希望交流）
改变值用利用block方法传值，不用delegate和通知的原因是感觉都太复杂了，通知还需要注册，但block的使用要注意避免循环引用导致内存没办法得到释放的问题。
在.h文件中定义block的别名便于使用

`typedef void (^CDZStringResultBlock)(NSString *string);`

.m中增加以下属性

```objectivec
@property (nonatomic,strong) NSArray *dataArray;
@property (nonatomic,strong) NSString *lastString;
@property (nonatomic,  copy) CDZStringResultBlock block;
```
实现Pickerview的Delegate和DataSource协议
`<UIPickerViewDelegate,UIPickerViewDataSource>`
实现Pickerview的Delegate和DataSource相关方法

```
#pragma mark - PickerDataSource
//返回picker列数
- (NSInteger)numberOfComponentsInPickerView:(UIPickerView *)pickerView{
    return 1;
}

//picker行数
- (NSInteger)pickerView:(UIPickerView *)pickerView numberOfRowsInComponent:(NSInteger)component{
    return self.dataArray.count;
}
```

```
#pragma mark - PickerDelegate
//返回每行高度
- (CGFloat)pickerView:(UIPickerView *)pickerView rowHeightForComponent:(NSInteger)component{
    return 44;
}

//滑动到当行进行的操作，这里把当行的数据回调
- (void)pickerView:(UIPickerView *)pickerView didSelectRow:(NSInteger)row inComponent:(NSInteger)component{
    self.block(self.dataArray[row]);
}

//要修改picker滚动里每行文字的值及相关属性，分割线等在此方法里设置
- (UIView *)pickerView:(UIPickerView *)pickerView viewForRow:(NSInteger)row forComponent:(NSInteger)component reusingView:(UIView *)view{
    //设置分割线的颜色,这里设为隐藏
    for(UIView *singleLine in pickerView.subviews){
        if (singleLine.frame.size.height < 1) {
            singleLine.backgroundColor = [UIColor clearColor];
        }
    }
    
    //设置文字的属性
    UILabel *genderLabel = [UILabel new];
    genderLabel.textAlignment = NSTextAlignmentCenter;
    genderLabel.text = self.dataArray[row];
    genderLabel.font = [UIFont systemFontOfSize:23.0];
    genderLabel.textColor = [UIColor blackColor];

    return genderLabel;
}
```
另外有一点要注意
`-(UIView *)pickerView:(UIPickerView *)pickerView viewForRow:(NSInteger)row forComponent:(NSInteger)component reusingView:(UIView *)view`
这个方法的优先级比
`- (NSString *)pickerView:(UIPickerView *)pickerView titleForRow:(NSInteger)row forComponent:(NSInteger)component`
这个方法优先级高，也就是说会覆盖后面那个方法里的设置，后面的方法只能确定每行返回的方法的值，没办法对字体大小，颜色，分割线等进行自定义，需要自定义就使用第一个方法。

### 2. 视图

视图上，由一个压黑背景的view上面加上一个包含确定和取消两个按钮和pickerview的containview组成。
先在最开始进行一些颜色，屏幕大小的宏定义和全局静态变量定义，方便使用

```
#define SCREEN_WIDTH ([UIScreen mainScreen].bounds.size.width)
#define SCREEN_HEIGHT ([UIScreen mainScreen].bounds.size.height)
#define BACKGROUND_BLACK_COLOR [UIColor colorWithRed:0.412 green:0.412 blue:0.412 alpha:0.7]
static const int pickerViewHeight = 228;
static const int toolBarHeight = 44;
```
进行view的布局，增加button点击动作等，pickerview要指定datasource和delegate为self

```
- (void)initView{
    UIView *containerView = [[UIView alloc]initWithFrame:CGRectMake(0, SCREEN_HEIGHT - pickerViewHeight, SCREEN_WIDTH, pickerViewHeight)];
    containerView.backgroundColor = [UIColor whiteColor];
    
    UIButton *btnOK = [[UIButton alloc] initWithFrame:CGRectMake(SCREEN_WIDTH -70, 5, 40, 30)];
    btnOK.titleLabel.font = [UIFont systemFontOfSize:18.0];
    [btnOK setTitle:@"确定" forState:UIControlStateNormal];
    [btnOK setTitleColor:[UIColor blueColor] forState:UIControlStateNormal];
    [btnOK addTarget:self action:@selector(pickerViewBtnOk:) forControlEvents:UIControlEventTouchUpInside];
    [containerView addSubview:btnOK];
    
    UIButton *btnCancel = [[UIButton alloc] initWithFrame:CGRectMake(30, 5, 40, 30)];
    btnCancel.titleLabel.font = [UIFont systemFontOfSize:18.0];
    [btnCancel setTitle:@"取消" forState:UIControlStateNormal];
    [btnCancel setTitleColor:[UIColor blueColor] forState:UIControlStateNormal];
    [btnCancel addTarget:self action:@selector(pickerViewBtnCancel:) forControlEvents:UIControlEventTouchUpInside];
    [containerView addSubview:btnCancel];
    
    UIPickerView *pickerView = [[UIPickerView alloc] initWithFrame:CGRectMake(0, 32, SCREEN_WIDTH, pickerViewHeight - toolBarHeight)];
    pickerView.backgroundColor = [UIColor whiteColor];
    pickerView.delegate = self;
    pickerView.dataSource = self;
    [containerView addSubview:pickerView];
    
    self.backgroundColor = BACKGROUND_BLACK_COLOR;
    [self addSubview:containerView];
}
```
### 3.响应按钮点击事件

很简单，见代码即可。取消的话把传进来的lastString传回去即可。

```
#pragma mark - event response
- (void)pickerViewBtnOk:(UIButton *)btn{
    [self removeFromSuperview];
}

- (void)pickerViewBtnCancel:(UIButton *)btn{
    self.block (self.lastString);
    [self removeFromSuperview];
}
```
### 4.封装成工厂方法

```
+ (void)showPickerInView:(UIView *)view
        withObjectsArray:(NSArray *)array
          withlastString:(NSString *)string
         withStringBlock:(CDZStringResultBlock)stringBlock{
    CDZPicker *pickerView = [[CDZPicker alloc]initWithFrame:view.bounds];
    pickerView.dataArray = array;
    pickerView.lastString = string;
    pickerView.block = stringBlock;
    pickerView.block(array[0]);//未滑动的话默认为第一个数据
    [pickerView initView];
    [view addSubview:pickerView];
}
```
### 最后

所有源码和[Demo](https://github.com/Nemocdz/CDZPickerViewDemo)
这是我第一次写技术类文章，虽然不是什么很深入的问题，但希望能分享给有需要的人。

如果您觉得有帮助,不妨给个star鼓励一下,欢迎关注&交流


























