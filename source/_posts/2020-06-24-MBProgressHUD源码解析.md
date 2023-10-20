---
title: MBProgressHUD源码解析
date: 2020-06-24 18:11:38
tags:
---

# MBProgressHUD

从入行以来,就经常听说,学习编程进步最快的方式,就是阅读优秀作品的源码...

那么,今天我来阅读一下MBProgressHUD这个库的源码...

查看一下文件,非常简单的俩个MBProgressHUD.h和MBProgressHUD.m文件

没什么说的,从MBProgressHUD.h文件开始看吧;

头文件首先声明了一个自定义的类 `MBBackgroundView` 和 一个协议 `MBProgressHUDDelegate`,自定义类放在这里声明一下,应该是因为`MBProgressHUD`里面使用到了`MBBackgroundView`类,且`MBProgressHUD`的声明与实现都放在了`MBBackgroundView`的前面,将`MBBackgroundView`的位置挪到比`MBProgressHUD`靠前的地方就不需要提前声明了,这个很简单,不需要多说了吧;而代理协议`MBProgressHUDDelegate`,对于做过iOS的开发人来说,应该说是再熟悉不过了...

接下来声明了一个供外部使用的变量`extern CGFloat const MBProgressMaxOffset;`,使用`extern`修饰了的变量能够在其他文件也可以访问到;往下查看头文件的时候,发现是为了给属性`offset`使用的,注释里面写到可以使用CGPointMake(0.f, MBProgressMaxOffset)来使HUD的位置处于底部边缘的中心位置

然后是4个自定义类型,`MBProgressHUDMode``MBProgressHUDAnimation``MBProgressHUDBackgroundStyle``MBProgressHUDCompletionBlock`;这四个类型也都比较简单,见名字大概就知道是什么意思了,简单提一下;
`MBProgressHUDMode`mode翻译过来叫(设备的)模式,方式,风格,样式...在这里取风格或者样式应该更加恰当;它有6种不同的样式:

* `MBProgressHUDModeIndeterminate `	//iOS系统原生的UIActivityIndicatorView
* `MBProgressHUDModeDeterminate `	//一个圆形的饼状进度视图样式
* `MBProgressHUDModeDeterminateHorizontalBar `	//一个水平的条状进度视图样式
* `MBProgressHUDModeAnnularDeterminate `	//圆环形的进度视图样式
* `MBProgressHUDModeCustomView `	//自定义视图样式
* `MBProgressHUDModeText ` //纯文字样式

`MBProgressHUDAnimation`动画类型

* `MBProgressHUDAnimationFade` //淡入淡出
* `MBProgressHUDAnimationZoom` //出现时放大,消失时缩小
* `MBProgressHUDAnimationZoomOut` //缩小
* `MBProgressHUDAnimationZoomIn` //放大

`MBProgressHUDBackgroundStyle`背景风格

* `MBProgressHUDBackgroundStyleSolidColor` //纯色
* `MBProgressHUDBackgroundStyleBlur` //模糊,毛玻璃效果

`MBProgressHUDCompletionBlock`是一个无返回值无参数的block

接来下是`MBProgressHUD`的正式声明了
首先是7个方法声明

前三个为类方法:

`+ (instancetype)showHUDAddedTo:(UIView *)view animated:(BOOL)animated;`

`+ (BOOL)hideHUDForView:(UIView *)view animated:(BOOL)animated;`

`+ (nullable MBProgressHUD *)HUDForView:(UIView *)view;`

后面为对象方法:

`- (instancetype)initWithView:(UIView *)view;`

`- (void)showAnimated:(BOOL)animated;`

`- (void)hideAnimated:(BOOL)animated;`

`- (void)hideAnimated:(BOOL)animated afterDelay:(NSTimeInterval)delay;`


每个方法的注释都很详细,不需要多介绍什么的,接下来看看属性

`@property (weak, nonatomic) id<MBProgressHUDDelegate> delegate;`
`@property (copy, nullable) MBProgressHUDCompletionBlock completionBlock;`
这两个属性都是用来回传HUD消失事件的,代理协议里面也只有一个HUD已经隐藏的方法

`@property (assign, nonatomic) NSTimeInterval graceTime;`这个属性很有意思,一开始我理解成下面那个属性`minShowTime`了,往下看到`minShowTime`之后,我才发现理解错了;这个属性的意思是:如果有graceTime,那么HUD在graceTime之后才显示.作用就是不让时间非常短的任务显示HUD(嗯,用处好像不是很大,我看了下我们项目,没有一个地方使用...)举个例子:我们一般会在网络请求发起之前显示HUD,在网络请求返回时隐藏HUD,在不考虑下面`minShowTime`属性的情况下,如果这个时间间隔非常短,就会出现显示了HUD瞬间就消失了的尴尬情况,那么这个时候,设置一个graceTime,对于那些时间非常短的异步任务就根本不会显示也没有必要显示HUD了

`@property (assign, nonatomic) NSTimeInterval minShowTime;`这个属性的作用就容易理解多了,最小显示时间;同样也是为了解决时间间隔很短的异步任务问题,你还察觉不到就结束了,那么这个HUD压根就看不见...所以加上这么一个时间

`@property (assign, nonatomic) BOOL removeFromSuperViewOnHide;`这个也是非常好理解,隐藏的时候是否从父视图移除

###### 下面的属性时跟外观(Appearance)相关的
`@property (assign, nonatomic) MBProgressHUDMode mode;` 样式
`@property (strong, nonatomic, nullable) UIColor *contentColor` 内容颜色
`@property (assign, nonatomic) MBProgressHUDAnimation animationType` 显示或隐藏时的动画类型
`@property (assign, nonatomic) CGPoint offset` 相对于视图中心的边框偏移量
`@property (assign, nonatomic) CGFloat margin` HUD边缘和HUD元素之间的间距
`@property (assign, nonatomic) CGSize minSize` HUD边框的最小尺寸
`@property (assign, nonatomic, getter = isSquare) BOOL square` 如果可能的话，强制HUD尺寸相等。
`@property (assign, nonatomic, getter=areDefaultMotionEffectsEnabled) BOOL defaultMotionEffectsEnabled` 当启用时，bezel center会受到设备加速计数据的轻微影响
###### 进度相关属性(Progress)
`@property (assign, nonatomic) float progress;` 进度指示器的进度,取值0.0~1.0,默认为0.0
###### 进度对象相关属性(ProgressObject)
`@property (strong, nonatomic, nullable) NSProgress *progressObject;` 不是太明白干什么的
###### 视图相关属性(Views)
`@property (strong, nonatomic, readonly) MBBackgroundView *bezelView;` 包含文本标签和指示器(或者自定义视图)的边框视图
`@property (strong, nonatomic, readonly) MBBackgroundView *backgroundView;` 覆盖整个HUD区域，放置在bezelView后面的视图
`@property (strong, nonatomic, nullable) UIView *customView;` 当HUD样式为MBProgressHUDModeCustomView的时候,显示的视图;视图应该实现intrinsicContentSize方法已显示正确的大小,为了得到最好的效果,建议使用37x37像素
`@property (strong, nonatomic, readonly) UILabel *label;` 显示在活动指示器下方的文本标签,HUD会自动调整大小以适应整个文本
`@property (strong, nonatomic, readonly) UILabel *detailsLabel;` 一个标签，其中包含一个可选的详细信息消息，显示在labelText消息下面。细节文本可以跨越多行。
`@property (strong, nonatomic, readonly) UIButton *button;` 放在标签下面的按钮。只有在添加target和action后才可见。

好了,MBProgressHUD的方法属性就这些了...
接下来看看MBProgressHUD头文件里,其他的类的声明;

首先是`MBRoundProgressView`这个应该是饼图样式时使用到的视图,只有4个属性
`@property (nonatomic, assign) float progress;` 进度
`@property (nonatomic, strong) UIColor *progressTintColor;` 指示器进度的颜色
`@property (nonatomic, strong) UIColor *backgroundTintColor;` 指示器背景色
`@property (nonatomic, assign, getter = isAnnular) BOOL annular;` 是否环形,不是很清楚什么效果...

然后是`MBBarProgressView`,这个应该是一个扁平的进度条视图
`@property (nonatomic, assign) float progress;` 进度
`@property (nonatomic, strong) UIColor *lineColor;` 条状边框线颜色
`@property (nonatomic, strong) UIColor *progressRemainingColor;` 条状背景色
`@property (nonatomic, strong) UIColor *progressColor;` 条状进度颜色

最后是`MBBackgroundView`,这个是背景视图
`@property (nonatomic) MBProgressHUDBackgroundStyle style;` 背景样式,毛玻璃或者纯色,iOS7和它之后都是毛玻璃效果,之前是纯色
`@property (nonatomic) UIBlurEffectStyle blurEffectStyle;` 模糊效果样式
`@property (nonatomic, strong) UIColor *color;` 背景色

以上就是MBProgressHUD.h文件的全部内容了,其实还有一部分,是遗弃的旧版本的代码就不看了

下面,我来看看.m文件源码,这才是重点啊






































