title: MBProgressHUD 源码分析
tags:
  - 源码分析
categories:
  - 三方开源库
author: leveltsui
date: 2017-11-08 22:26:00
---
在项目中经常会使用`MBProgressHUD`来实现弹窗提醒，所有来分析下`MBProgressHUD`这个三方库的代码。所分析的源码版本号为1.0.0。
***
这篇总结主要分三个部分来介绍分析这个框架：
- 代码结构
- 方法调用流程图
- 方法内部实现
***
##### 代码结构
###### 类图

![MBProgressHUD.png](http://upload-images.jianshu.io/upload_images/117999-5da3f0778692f17f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#####  核心API
###### 属性
 ```objc
/*
 * 用来推迟HUD的显示，避免HUD显示时间过短，出现一闪而逝的情况，默认值为0。
 */
@property (assign, nonatomic) NSTimeInterval graceTime;

/**
 *  HUD最短显示时间，单位为s，默认值为0。 
 */
@property (assign, nonatomic) NSTimeInterval minShowTime;

/**
 * HUD隐藏时，将其从父视图上移除 。默认值为NO 
 */
@property (assign, nonatomic) BOOL removeFromSuperViewOnHide; 

/** 
 * HUD显示类型，默认为 MBProgressHUDModeIndeterminate.
 */
@property (assign, nonatomic) MBProgressHUDMode mode; 
 ```
 ######  类方法
```objc
/**
 * 创建HUD,添加到提供的视图上并显示
 */
+ (instancetype)showHUDAddedTo:(UIView *)view animated:(BOOL)animated;

/**
 * 找到最上层的HUD,并隐藏。
 */
+ (BOOL)hideHUDForView:(UIView *)view animated:(BOOL)animated;

/**
 * 在传入的View上找到最上层的HUD并隐藏此HUD
 */
+ (nullable MBProgressHUD *)HUDForView:(UIView *)view;
```
 ###### 实例方法
```objc
/**
 * 构造函数，用来初始化HUD
 */
- (instancetype)initWithView:(UIView *)view;

/** 
 * 显示HUD
 */
- (void)showAnimated:(BOOL)animated;

/** 
 * 隐藏HUD
 */
- (void)hideAnimated:(BOOL)animated;
```
 ##### 方法调用流程图 
从`MBProgressHUD`提供的主要接口可以看出，主要有显示HUD和隐藏HUD这两个功能，一步步追溯，得出的方法调用流程图如下：
![MBProgressHUD流程图.png](http://upload-images.jianshu.io/upload_images/117999-4c522bd388a1a65a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 方法内部实现
方法的内部实现主要从两个方面来分析，显示HUD和隐藏HUD。
###### 显示HUD
首先是MBProgressHUD的构造方法
```objc
+ (instancetype)showHUDAddedTo:(UIView *)view animated:(BOOL)animated {
    //初始化MBProgressHUD
    MBProgressHUD *hud = [[self alloc] initWithView:view];
    hud.removeFromSuperViewOnHide = YES;
    [view addSubview:hud];
    [hud showAnimated:animated];
    [[UINavigationBar appearance] setBarTintColor:nil];
    return hud;
}
```
首先进入`- (id)initWithView:(UIView *)view`方法，再进入`- (instancetype)initWithFrame:(CGRect)frame`方法，最后调用`- (void)commonInit`方法，进行属性的初始化和添加子视图。
```objc
- (void)commonInit {
    // Set default values for properties
    _animationType = MBProgressHUDAnimationFade;
    _mode = MBProgressHUDModeIndeterminate;
    _margin = 20.0f;
    _opacity = 1.f;
    _defaultMotionEffectsEnabled = YES;

    // Default color, depending on the current iOS version
    BOOL isLegacy = kCFCoreFoundationVersionNumber < kCFCoreFoundationVersionNumber_iOS_7_0;
    _contentColor = isLegacy ? [UIColor whiteColor] : [UIColor colorWithWhite:0.f alpha:0.7f];
    // Transparent background
    self.opaque = NO;
    self.backgroundColor = [UIColor clearColor];
    // Make it invisible for now
    self.alpha = 0.0f;
    self.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    self.layer.allowsGroupOpacity = NO;
    //添加子视图
    [self setupViews];
    //更新指示器
    [self updateIndicators];
    [self registerForNotifications];
}
```
添加子视图都是常见的方式，让视图跟随陀螺仪运动，这个之前没有接触过，后续需要了解下。
```objc
- (void)updateBezelMotionEffects {
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= 70000 || TARGET_OS_TV
    MBBackgroundView *bezelView = self.bezelView;
    if (![bezelView respondsToSelector:@selector(addMotionEffect:)]) return;

    if (self.defaultMotionEffectsEnabled) {
        CGFloat effectOffset = 10.f;
        UIInterpolatingMotionEffect *effectX = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.x" type:UIInterpolatingMotionEffectTypeTiltAlongHorizontalAxis];
        effectX.maximumRelativeValue = @(effectOffset);
        effectX.minimumRelativeValue = @(-effectOffset);

        UIInterpolatingMotionEffect *effectY = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.y" type:UIInterpolatingMotionEffectTypeTiltAlongVerticalAxis];
        effectY.maximumRelativeValue = @(effectOffset);
        effectY.minimumRelativeValue = @(-effectOffset);

        UIMotionEffectGroup *group = [[UIMotionEffectGroup alloc] init];
        group.motionEffects = @[effectX, effectY];

        [bezelView addMotionEffect:group];
    } else {
        NSArray *effects = [bezelView motionEffects];
        for (UIMotionEffect *effect in effects) {
            [bezelView removeMotionEffect:effect];
        }
    }
#endif
}
```
再主要看下更新指示器的代码。
```objc
- (void)updateIndicators { 
    UIView *indicator = self.indicator;
    BOOL isActivityIndicator = [indicator isKindOfClass:[UIActivityIndicatorView class]];
    BOOL isRoundIndicator = [indicator isKindOfClass:[MBRoundProgressView class]];

    MBProgressHUDMode mode = self.mode;
    //菊花动画
    if (mode == MBProgressHUDModeIndeterminate) {
        if (!isActivityIndicator) {
            // Update to indeterminate indicator
            [indicator removeFromSuperview];
            indicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleWhiteLarge];
            [(UIActivityIndicatorView *)indicator startAnimating];
            [self.bezelView addSubview:indicator];
        }
    }
    //水平进度条动画
    else if (mode == MBProgressHUDModeDeterminateHorizontalBar) {
        // Update to bar determinate indicator
        [indicator removeFromSuperview];
        indicator = [[MBBarProgressView alloc] init];
        [self.bezelView addSubview:indicator];
    }
    //圆形进度动画
    else if (mode == MBProgressHUDModeDeterminate || mode == MBProgressHUDModeAnnularDeterminate) {
        if (!isRoundIndicator) {
            // Update to determinante indicator
            [indicator removeFromSuperview];
            indicator = [[MBRoundProgressView alloc] init];
            [self.bezelView addSubview:indicator];
        }
        //环形动画
        if (mode == MBProgressHUDModeAnnularDeterminate) {
            [(MBRoundProgressView *)indicator setAnnular:YES];
        }
    }
    //自定义动画
    else if (mode == MBProgressHUDModeCustomView && self.customView != indicator) {
        // Update custom view indicator
        [indicator removeFromSuperview];
        indicator = self.customView;
        [self.bezelView addSubview:indicator];
    }
    //只显示文本
    else if (mode == MBProgressHUDModeText) {
        [indicator removeFromSuperview];
        indicator = nil;
    }
    indicator.translatesAutoresizingMaskIntoConstraints = NO;
    self.indicator = indicator;

    if ([indicator respondsToSelector:@selector(setProgress:)]) {
        [(id)indicator setValue:@(self.progress) forKey:@"progress"];
    }

    [indicator setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisHorizontal];
    [indicator setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisVertical];

    [self updateViewsForColor:self.contentColor];
    [self setNeedsUpdateConstraints];
}
```
在这个方法中，主要是根据显示的模式，将不同的`indicator`视图赋值给`indicator`属性。更新完指示器后，就是开始将视图显示在界面上。调用的是`- (void)showAnimated:(BOOL)animated`方法。
```objc

- (void)showAnimated:(BOOL)animated {
    //保证当前线程是主线程
    MBMainThreadAssert();
    [self.minShowTimer invalidate];
    self.useAnimation = animated;
    self.finished = NO;
    // 如果设置了宽限时间，则推迟HUD的显示
    if (self.graceTime > 0.0) {
        NSTimer *timer = [NSTimer timerWithTimeInterval:self.graceTime
                                                 target:self
                                               selector:@selector(handleGraceTimer:)
                                               userInfo:nil
                                                repeats:NO];
        //默认把你的Timer以NSDefaultRunLoopMode添加到MainRunLoop上，而当当前视图在滚动时，当前的MainRunLoop是处于UITrackingRunLoopMode的模式下，在这个模式下，是不会处理NSDefaultRunLoopMode的消息，要想在scrollView滚动的同时Timer也执行的话，我们需要将Timer以NSRunLoopCommonModes的模式注册到当前RunLoop中.
        [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        self.graceTimer = timer;
    } 
    // ... otherwise show the HUD immediately
    else {
        [self showUsingAnimation:self.useAnimation];
    }
}
```
在`- (void)showAnimated:(BOOL)animated`方法中，主要做的是判断是否设置了推迟显示HUD的时间，如果设置了，就推迟设置的时间再显示。最后，执行`- (void)showUsingAnimation:(BOOL)animated`方法。
```objc

- (void)showUsingAnimation:(BOOL)animated {
    // Cancel any previous animations
    [self.bezelView.layer removeAllAnimations];
    [self.backgroundView.layer removeAllAnimations];

    // Cancel any scheduled hideDelayed: calls
    [self.hideDelayTimer invalidate];
    //记录当前显示的时间，在HUD隐藏时，比较HUD显示到HUD隐藏之间的间隔与最小显示时间，
    //如果小于，继续显示，直到显示时间等于最小显示时间，再隐藏HUD
    self.showStarted = [NSDate date];
    self.alpha = 1.f;

    // Needed in case we hide and re-show with the same NSProgress object attached.
    //好像是通过这个去刷新进度，这个需要再查下。
    [self setNSProgressDisplayLinkEnabled:YES];

    if (animated) {
        [self animateIn:YES withType:self.animationType completion:NULL];
    } else {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
        self.bezelView.alpha = self.opacity;
#pragma clang diagnostic pop
        self.backgroundView.alpha = 1.f;
    }
}
```
最后执行`- (void)animateIn:(BOOL)animatingIn withType:(MBProgressHUDAnimation)type completion:(void(^)(BOOL finished))completion`方法，这个方法显示和隐藏均会调用。
```objc
//这个方法主要对self.bezelView视图进行动画
- (void)animateIn:(BOOL)animatingIn withType:(MBProgressHUDAnimation)type completion:(void(^)(BOOL finished))completion {
    // Automatically determine the correct zoom animation type
    if (type == MBProgressHUDAnimationZoom) {
        type = animatingIn ? MBProgressHUDAnimationZoomIn : MBProgressHUDAnimationZoomOut;
    }

    CGAffineTransform small = CGAffineTransformMakeScale(0.5f, 0.5f);
    CGAffineTransform large = CGAffineTransformMakeScale(1.5f, 1.5f);

    // Set starting state
    UIView *bezelView = self.bezelView;
    if (animatingIn && bezelView.alpha == 0.f && type == MBProgressHUDAnimationZoomIn) {
        bezelView.transform = small;
    } else if (animatingIn && bezelView.alpha == 0.f && type == MBProgressHUDAnimationZoomOut) {
        bezelView.transform = large;
    }

    // 使用动画
    dispatch_block_t animations = ^{
        if (animatingIn) {
            bezelView.transform = CGAffineTransformIdentity;
        } else if (!animatingIn && type == MBProgressHUDAnimationZoomIn) {
            bezelView.transform = large;
        } else if (!animatingIn && type == MBProgressHUDAnimationZoomOut) {
            bezelView.transform = small;
        }
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
        bezelView.alpha = animatingIn ? self.opacity : 0.f;
#pragma clang diagnostic pop
        self.backgroundView.alpha = animatingIn ? 1.f : 0.f;
    };

    // Spring animations are nicer, but only available on iOS 7+
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= 70000 || TARGET_OS_TV
    if (kCFCoreFoundationVersionNumber >= kCFCoreFoundationVersionNumber_iOS_7_0) {
        [UIView animateWithDuration:0.3 delay:0. usingSpringWithDamping:1.f initialSpringVelocity:0.f options:UIViewAnimationOptionBeginFromCurrentState animations:animations completion:completion];
        return;
    }
#endif
    [UIView animateWithDuration:0.3 delay:0. options:UIViewAnimationOptionBeginFromCurrentState animations:animations completion:completion];
}
```
从代码可以看出，这里只是对指示器的父视图做了放大缩小的动画。
###### 隐藏HUD
```objc

+ (BOOL)hideHUDForView:(UIView *)view animated:(BOOL)animated {
    //获取当前显示的hud,如果存在，当前隐藏时，将其从父视图移除
    MBProgressHUD *hud = [self HUDForView:view];
    if (hud != nil) {
        hud.removeFromSuperViewOnHide = YES;
        [hud hideAnimated:animated];
        return YES;
    }
    return NO;
}
```
在这个方法的执行过程中，调用`- (void)hideAnimated:(BOOL)animated `方法。
```objc
 - (void)hideAnimated:(BOOL)animated {
    MBMainThreadAssert();
    [self.graceTimer invalidate];
    self.useAnimation = animated;
    self.finished = YES;
    // 如果设置了最小显示时间，计算HUD显示时长，
    // 如果HUD显示时长小于最小显示时间，延迟显示
    if (self.minShowTime > 0.0 && self.showStarted) {
        NSTimeInterval interv = [[NSDate date] timeIntervalSinceDate:self.showStarted];
        if (interv < self.minShowTime) {
            NSTimer *timer = [NSTimer timerWithTimeInterval:(self.minShowTime - interv) target:self selector:@selector(handleMinShowTimer:) userInfo:nil repeats:NO];
            [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
            self.minShowTimer = timer;
            return;
        } 
    }
    // ... otherwise hide the HUD immediately
    [self hideUsingAnimation:self.useAnimation];
}
```
`- (void)hideAnimated:(BOOL)animated `方法中，主要做的是判断是否需要推迟隐藏HUD，最后调用`- (void)hideUsingAnimation:(BOOL)animated`方法，
```objc

- (void)hideUsingAnimation:(BOOL)animated {
    //判断是否需要动画效果，如无，则直接隐藏
    if (animated && self.showStarted) {
        self.showStarted = nil;
        //跟显示HUD差不多，只是指示器父视图没有做放大缩小的动画
        [self animateIn:NO withType:self.animationType completion:^(BOOL finished) {
            [self done];
        }];
    } else {
        self.showStarted = nil;
        self.bezelView.alpha = 0.f;
        self.backgroundView.alpha = 1.f;
        [self done];
    }
}
```
最后，调用`- (void)done`方法。这个方法主要负责属性的释放和隐藏完成回调的处理。
```objc
- (void)done {
    // Cancel any scheduled hideDelayed: calls
    [self.hideDelayTimer invalidate];
    //指示进度的显示问题，后续还需再补充
    [self setNSProgressDisplayLinkEnabled:NO];

    if (self.hasFinished) {
        self.alpha = 0.0f;
        if (self.removeFromSuperViewOnHide) {
            [self removeFromSuperview];
        }
    }
    MBProgressHUDCompletionBlock completionBlock = self.completionBlock;
    if (completionBlock) {
        completionBlock();
    }
    id<MBProgressHUDDelegate> delegate = self.delegate;
    if ([delegate respondsToSelector:@selector(hudWasHidden:)]) {
        [delegate performSelector:@selector(hudWasHidden:) withObject:self];
    }
}
``` 
##### 总结
从代码来看，`MBProgressHUD`这个三方库有几个地方值借鉴：
- `graceTime`和`minShowTime`，在开发的时候会出现显示HUD后，存在缓存或者网速较好时，HUD显示到HUD隐藏的时间较短，界面出现闪动的情况，这时，就可以通过设置`graceTime`和`minShowTime`来处理，达到更好的用户体验。 这个在封装弹窗控件时，可以参考。