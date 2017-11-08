title: iOS布局与Masnory使用实践
author: leverTsui
author_id: leverTsui
language: zh-Hans
date: 2017-11-06 18:46:28 
tags:
  - 自动布局 AutoLayout Masnory
categories:
  - iOS 自动布局 
---
##### 前言
`UI`布局对于`iOS`开发者来说并不陌生，在`iOS6`之前，大家都是通过`UI`控件的`Frame`属性和`Autoresizing Mask`来进行`UI`布局的（简称为手动布局）。`AutoLayout`则是苹果公司在`iOS6`推出的一种基于约束的，描述性的布局系统（简称为自动布局），这里主要从四个方面来阐述iOS布局及实践。
- 手动布局和自动布局
- `AutoLayout`原理
- `AutoLayout`的性能
- `Masnory`的使用

首先对手动布局和自动布局做一个简单的介绍：
##### 手动布局和自动布局
- 手动布局：指的是通过直接修改视图的`frame`属性的方式对界面进行布局。 
> 对于`IOS`的`app`开发者来说，不会像`Android`开发者一样为很多的屏幕尺寸来做界面适配，因此手动调整 `frame`的方式来布局也能工作良好。但是还是会有一些问题，如设备发生旋转、适配`ipad`等，并且保证视图原来之间的相对关系，则以上的方法都是无法解决的。如果要做这些适配，在`AutoLayout`未出来之前需要编写大量的代码，并且花费大量的调试适配时间。 

- 自动布局：指的是使用`AutoLayout`的方式对界面进行布局。

> `AutoLayout` 是苹果本身提倡的技术，在大部分情况下也能很好的提升开发效率，但是 `AutoLayout `对于复杂视图来说常常会产生严重的性能问题。随着视图数量的增长，`AutoLayout` 带来的 `CPU` 消耗会呈指数级上升。 如果对界面流畅度要求较高（如微博界面），可以通过提前计算好布局，在需要时一次性调整好对应属性 ，或者使用 `ComponentKit`、`AsyncDisplayKit` 等框架来处理界面布局。

下面，我们来分析下 AutoLayout的原理。
##### AutoLayout的原理 
这里通过使用`Masonry`来进行布局，从而来分析`AutoLayout`的原理，先简要了解下`Masonry`。
`Masonry`是一个轻量级的布局框架，拥有自己的描述语法，采用更优雅的链式语法封装自动布局，简洁明了，并具有高可读性，而且同时支持 `iOS` 和 `Max OS X`。
`Masnory`支持的常用属性如下：
```objc
@property (nonatomic, strong, readonly) MASConstraint *left;     //左侧
@property (nonatomic, strong, readonly) MASConstraint *top;      //上侧
@property (nonatomic, strong, readonly) MASConstraint *right;   //右侧
@property (nonatomic, strong, readonly) MASConstraint *bottom;   //下侧
@property (nonatomic, strong, readonly) MASConstraint *leading;  //首部
@property (nonatomic, strong, readonly) MASConstraint *trailing;  //首部
@property (nonatomic, strong, readonly) MASConstraint *width;    //宽
@property (nonatomic, strong, readonly) MASConstraint *height;   //高
@property (nonatomic, strong, readonly) MASConstraint *centerX;  //横向中点
@property (nonatomic, strong, readonly) MASConstraint *centerY;  //纵向中点
@property (nonatomic, strong, readonly) MASConstraint *baseline; //文本基线 
```
**其中`leading`与`left`，`trailing`与`right` 在正常情况下是等价的，但是当一些布局是从右至左时(比如阿拉伯语) 则会对调。**
同时，在`Masonry`中能够添加`AutoLayout`约束有三个函数：
```objc
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *make))block;//只负责新增约束` AutoLayout`不能同时存在两条针对于同一对象的约束,否则会报错
- (NSArray *)mas_updateConstraints:(void(^)(MASConstraintMaker *make))block;//针对上面的情况 会更新在block中出现的约束 不会导致出现两个相同约束的情况
- (NSArray *)mas_remakeConstraints:(void(^)(MASConstraintMaker *make))block;//则会清除之前的所有约束 仅保留最新的约束
```
我们在代码中，经常会使用到`equalTo`和`mas_equalTo`，那它们的区别是什么呢？从代码中找到他们的定义如下：
```objc
#define mas_equalTo(...)                 equalTo(MASBoxValue((__VA_ARGS__)))
...
#define MASBoxValue(value) _MASBoxValue(@encode(__typeof__((value))), (value))
```
可以看到 `mas_equalTo`只是对其参数进行了一个`BOX`操作(装箱) ，所支持的类型，除了`NSNumber`支持的那些数值类型之外，还支持`CGPoint`，`CGSize`和`UIEdgeInsets`类型。
下面，我们通过一个例子，一步步来看下界面是怎么布局的，代码如下：
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor blackColor];
    
    UIView *v1 = [[UIView alloc] init];
    v1.backgroundColor = [UIColor orangeColor];
    [v1 showPlaceHolder];
    
    UIView *v2 = [[UIView alloc] init];
    v2.backgroundColor = [UIColor orangeColor];
    [v2 showPlaceHolder];
    
    UIView *v3 = [[UIView alloc] init];
    v3.backgroundColor = [UIColor orangeColor];
    [v3 showPlaceHolder];
    
    [self.view addSubview:v1];
    [self.view addSubview:v2];
    [self.view addSubview:v3];
    
    [v1 mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.mas_equalTo(100);
        make.leading.mas_equalTo(100);
        make.width.mas_equalTo(70);
        make.height.mas_equalTo(65);
    }];
    
    [v2 mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(v1.mas_top);
        make.leading.mas_equalTo(v1.mas_trailing).offset(20);
        make.width.equalTo(v1.mas_width);
        make.height.equalTo(v1.mas_height);
    }];
    
    [v3 mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(v1.mas_bottom).offset(20);
        make.leading.equalTo(v1.mas_leading);
        make.trailing.equalTo(v2.mas_trailing);
        make.height.equalTo(v1.mas_height);
    }];
} 
```
界面运行结果如下图：
![CD52E302-FAFD-4D6E-9DFF-F5DB44C6098B.png](http://upload-images.jianshu.io/upload_images/5835116-10acd4fa0c0e292c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下面，我们将界面中的左上角的视图视为视图1，右上角的视图视为视图2，底部视图视为视图3，使用`x1、y1、m1、n1`来标识视图1的`left`、`top`、`width`和`height`，以此类推。
通过以上举例抽象出自动布局数学公式： 
 ![1C7344E7-1ED1-421A-B84E-ACBD70F98859.png](http://upload-images.jianshu.io/upload_images/5835116-6ceec51e08a7f571.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将以上等式变形为：

![8AAD54BE-0157-4591-A117-0094F69BE6E7.png](http://upload-images.jianshu.io/upload_images/5835116-101854d5027207fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时，以上方程组，大家肯定很熟悉了，也就是《线性代数》中的线性方程组，现在将以上线性方程组抽象为：

![B5E84F57-07DE-4A15-9D06-3D0ADD7E6EBD.png](http://upload-images.jianshu.io/upload_images/5835116-665ec9939aa6a902.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图表示“等式”方程组，那么是否还可以继续抽象？也就是说上述方程组能否完全表示未知元素之间与已知元素之间的关系，显然还不全面，因为还有（<,>,<=,>=）不等关系，因此将“=”等号抽象为关系"R",在数学上关系R也就包括了“=”,"<",">","<=",">="等关系。上述线程方程组变形为：（实质上，AutoLayout中所有的约束确实都是用数学关系式y R ax + b描述）

![9A877277-5DEB-437C-AEF6-D6530AB6FE6F.png](http://upload-images.jianshu.io/upload_images/5835116-8ae25f0ead6f4c29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在已经将自动布局一步步抽象为数学公式，那么对视图的布局其实就是对线性方程组的求解。线性方程组解的情况有三种，实质上也对应着自动布局对视图的三种布局方案:
 - 唯一解：所有方程中的未知数能够解出唯一解。 充分约束：给一个视图添加的约束必须是充分的，才能正确布局一个视图；
 - 多个解：未知数不能求解出准确的唯一解，即未知数可能存在多个或者无限个解满足线性方程组。 欠约束：给视图所添加的约束不能够充分的表达视图的准确位置，在这种情况下自动布局会随意给视图一个布局方案，也就是自动布局中视图不能够正确布局或者视图丢失的情况。 
 - 无解：不存在满足线性方程组的解。 冲突约束：给视图添加的约束表达视图布局出现了冲突，比如同时满足同一个视图宽度即为100又为200，这是不可能存在的。此时程序会出现崩溃。

通过以上描述，将`AutoLayout`系统的作用描述如图所示：

![FBB5F53F-0B23-48B2-B150-39B405BFB335.png](http://upload-images.jianshu.io/upload_images/5835116-55978c66c8eea66b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
##### `AutoLayout`的性能

从`AutoLayout`的原理，我们可以得出布局系统最后仍然需要通过`frame`来进行布局，相比原有的布局系统加入了从约束计算 出`frame` 的过程,那么这个过程对性能是否会影响呢？
你可以在 [**这里**](https://github.com/hua16/summary) 找到这次对 `Layout` 性能测量使用的代码。
代码分别使用` Auto Layout `、嵌套视图层级中使用 `Auto Layout `和` frame `对 `N` 个视图进行布局，测算其运行时间。

对视图数量在 1~35 之间布局时间进行测量，结果如下：


![视图数量范围为 1~35.png](http://upload-images.jianshu.io/upload_images/117999-045780ced38306d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


对视图数量在 10~500 之间布局时间进行测量，结果如下：

![视图数量范围为 10~500.png](http://upload-images.jianshu.io/upload_images/117999-4a5f70dd55e894d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
从上述的测试数据可以看出，**使用`frame`、`AutoLayout`和嵌套视图层级中使用 `Auto Layout`进行布局、对应的视图数量分别为`50`个、`6`个和`12`个，所需要的时间就会在 `16.67 ms `左右。**,而想要让 iOS 应用的视图保持 60 FPS 的刷新频率，我们必须在 1/60 = 16.67 ms 之内完成包括布局、绘制以及渲染等操作。
综上所述，虽然说 `Auto Layout` 为开发者在多尺寸布局上提供了遍历，而且支持跨越视图层级的约束，但是由于其实现原理导致其时间复杂度为**多项式时间**，其性能损耗是仅使用 `frame` 的十几倍，所以在处理庞大的 `UI `界面时表现差强人意。 

##### `Masnory`的使用
下面，我们通过4个实例，来了解下`Masnory`的使用。
- ######case 1: 并排显示两个`label`，宽度由内容决定。父视图宽度不够时，优先显示右边`label`的内容。

在默认情况下，我们没有设置各个布局的优先级，那么他就会优先显示左边的`label`，左边的完全显示后剩余的空间都是右边的`label`，如果整个空间宽度都不够左边的`label`的话，那么右边的`label`就没有显示的机会了。
如果我们现在的需求是优先显示右边的`label`，左边的`label`内容超出的省略，这时就需要我们调整约束的优先级了。
`UIView`中关于`Content Hugging` 和` Content Compression Resistance`的方法有：
```objc
- (UILayoutPriority)contentHuggingPriorityForAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);
- (void)setContentHuggingPriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);

- (UILayoutPriority)contentCompressionResistancePriorityForAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);
- (void)setContentCompressionResistancePriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);
```
那么这两个东西到底是什么呢？可以这样形象的理解一下：
- `contentHugging`: 抱住使其在“内容大小”的基础上不能继续变大，这个属性的优先级越高，就要越“抱紧”视图里面的内容。也就是视图的大小不会随着父视图的扩大而扩大。
- `contentCompression`: 撑住使其在在其“内容大小”的基础上不能继续变小,这个属性的优先级越高，越不“容易”被压缩。也就是说，当整体的空间装不下所有的视图时，`Content Compression Resistance`优先级越高的，显示的内容越完整。
这两个属性分别可以设置水平方向和垂直方向上的，而且一个默认优先级是250， 一个默认优先级是750. 因为这两个很有可能与其他Constraint冲突，所以优先级较低。

```objc
static const UILayoutPriority UILayoutPriorityRequired NS_AVAILABLE_IOS(6_0) = 1000; // A required constraint.  Do not exceed this.
static const UILayoutPriority UILayoutPriorityDefaultHigh NS_AVAILABLE_IOS(6_0) = 750; // This is the priority level with which a button resists compressing its content.
static const UILayoutPriority UILayoutPriorityDefaultLow NS_AVAILABLE_IOS(6_0) = 250; // This is the priority level at which a button hugs its contents horizontally.
static const UILayoutPriority UILayoutPriorityFittingSizeLevel NS_AVAILABLE_IOS(6_0) = 50; 
```

```objc
- (void)layoutPageSubViews {
    
    [self.leftLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(self.contentView1.mas_top).with.offset(5);
        make.left.equalTo(self.contentView1.mas_left).with.offset(2);
        make.height.equalTo(@40);
    }];
    
    [self.rightLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.leftLabel.mas_right).with.offset(2);
        make.top.equalTo(self.contentView1.mas_top).with.offset(5);
        make.right.lessThanOrEqualTo(self.contentView1.mas_right).with.offset(-2);
        make.height.equalTo(@40);

    }];
    
    [self.leftLabel setContentHuggingPriority:UILayoutPriorityRequired
                               forAxis:UILayoutConstraintAxisHorizontal];
    [self.leftLabel setContentCompressionResistancePriority:UILayoutPriorityDefaultLow
                                             forAxis:UILayoutConstraintAxisHorizontal];
    
    [self.rightLabel setContentHuggingPriority:UILayoutPriorityRequired
                               forAxis:UILayoutConstraintAxisHorizontal];
    [self.rightLabel setContentCompressionResistancePriority:UILayoutPriorityRequired
                                             forAxis:UILayoutConstraintAxisHorizontal];
}
```
- ######case 2: 四个`ImageView`整体居中，可以任意显示、隐藏。

![blog_autolayout_example_with_masonry_3.png](http://upload-images.jianshu.io/upload_images/5835116-1334685bb89094f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面的四个`Switch`控件分别控制上面对应位置的图片是否显示。
> 分析:首先就是整体居中，为了实现这个，最简单的办法就是将四个图片“装进”一个**容器View**里面，然后让这个容器`View`在整个页面中居中即可。这样就不用控制每个图片的居中效果了。
然后就是显示与隐藏。在这里我直接控制图片`ImageView`的宽度，宽度为0的时候不就“隐藏”了吗。

具体代码如下：

```objc
- (void)layoutPageSubViews {
    
    [self.containerView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.height.mas_equalTo(IMAGE_SIZE);
        make.centerX.equalTo(self.view.mas_centerX);
        make.top.equalTo(self.view.mas_top).offset(200);
    }];
    
    //分别设置每个imageView的宽高、左边、垂直中心约束，注意约束的对象
    //每个View的左边约束和左边的View的右边相等
    __block UIView *lastView = nil;
    __block MASConstraint *widthConstraint = nil;
    NSUInteger arrayCount = self.imageViews.count;
    [self.imageViews enumerateObjectsUsingBlock:^(UIView *view, NSUInteger idx, BOOL *stop) {
        [view mas_makeConstraints:^(MASConstraintMaker *make) {
            make.left.equalTo(lastView ? lastView.mas_right : view.superview.mas_left);
            make.centerY.equalTo(view.superview.mas_centerY);
            if (idx == arrayCount - 1) {
                make.right.equalTo(view.superview.mas_right);
            }
            
            widthConstraint = make.width.mas_equalTo(IMAGE_SIZE);
            make.height.mas_equalTo(IMAGE_SIZE);
            
            [self.widthConstraints addObject:widthConstraint];
            lastView = view;
        }];
    }];
}

#pragma mark - event response
//点击switch按钮，如果打开，对应视图的宽约束设置为32，否则，设置为0
- (IBAction)showOrHideImage:(UISwitch *)sender {
    NSUInteger index = (NSUInteger) sender.tag;
    MASConstraint *width = self.widthConstraints[index];

    if (sender.on) {
        width.mas_equalTo(IMAGE_SIZE);
    } else {
        width.mas_equalTo(0);
    }
}
```
- #####case 3: 子视图的宽度始终是父视图的四分之三（或者任意百分比）
```objc

  //宽度为父view的宽度的四分之三 
[subView mas_makeConstraints:^(MASConstraintMaker *make) {
        //上下左贴边
        make.left.equalTo(_containerView.mas_left);
        make.top.equalTo(_containerView.mas_top);
        make.bottom.equalTo(_containerView.mas_bottom);
        //宽度为父view的宽度的一半
        make.width.equalTo(_containerView.mas_width).multipliedBy(0.75);
    }];
```
- #####case 4 给同一个属性添加多重约束，实现复杂关系

```objc

- (void)layoutPageSubviews {
    
    [self.greenLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.centerY.equalTo(self.containerView);
        make.right.lessThanOrEqualTo(self.containerView);
        make.left.greaterThanOrEqualTo(self.containerView.mas_right).multipliedBy((CGFloat)(1.0f / 3.0f));
        for (UILabel *label in self.leftLabels) {
            make.left.greaterThanOrEqualTo(label.mas_right).offset(8);
        }
    }];
    
    [self.greenLabel setContentCompressionResistancePriority:UILayoutPriorityRequired forAxis:UILayoutConstraintAxisHorizontal];
}
``` 
##### 总结
通过上述分析，我们可以发现：
- `AutoLayout`的原理就是对线性方程组或者不等式的求解，最终使用`frame`来绘制视图；
-  使用`AutoLayout`进行布局时， 由于其实现原理导致其时间复杂度为多项式时间，其性能损耗是仅使用 frame 的十几倍，所以在处理庞大的 UI界面时表现差强人意。