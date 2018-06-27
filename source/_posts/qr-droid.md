title: iOS二维码识别/二维码生成
tags:
  - 二维码 图片识别
categories:
  - iOS 开发
author: leveltsui
date: 2017-11-03 20:42:00
---
#### 前言
之前做过一个关于二维码的组件，已发布，现总结下。
开发的`APP`所需支持的最低版本为`8.0`，最初的方案为扫描使用苹果自带的`API`实现扫一扫的功能、使用`ZXing`识别从相册或别人转发的二维码图片。但发现`ZXing`识别从相册中来的图片性能很差，很多图片识别不了，且耗时较长，遂使用`ZBar`来实现识别从相册或别人转发的二维码图片。 
这个组件重要实现了三个功能，扫一扫识别二维码图片、长按图片识别二维码图片和生成二维码图片。
首先来看下扫一扫识别二维码图片的代码实现：

#### 功能实现
##### 扫一扫识别二维码图片 
```objc
- (void)initCapture {
    AVCaptureDevice* inputDevice =
    [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo]; 
    [inputDevice lockForConfiguration:nil];
    if ([inputDevice hasTorch]){
        inputDevice.torchMode = AVCaptureTorchModeAuto;
    }
    AVCaptureFocusMode foucusMode = AVCaptureFocusModeContinuousAutoFocus;
    if ([inputDevice isFocusModeSupported:foucusMode]) {
        inputDevice.focusMode = foucusMode;
    }
    [inputDevice unlockForConfiguration];
    
    AVCaptureDeviceInput *captureInput =
    [AVCaptureDeviceInput deviceInputWithDevice:inputDevice error:nil];
    
    if (!captureInput) {
        //支持的最低版本为iOS8
        UIAlertController *alterVC = [UIAlertController alertControllerWithTitle:MUIQRCodeLocalizedString(@"ScanViewController_system_tip") message:MUIQRCodeLocalizedString(@"ScanViewController_camera_permission") preferredStyle:UIAlertControllerStyleAlert];
        UIAlertAction *confirmAction = [UIAlertAction actionWithTitle:MUIQRCodeLocalizedString(@"ScanViewController_yes") style:UIAlertActionStyleDefault handler:nil];
        [alterVC addAction:confirmAction];
        [self presentViewController:alterVC animated:YES completion:nil];
        [self.activityView stopAnimating];
        [self onVideoStart:nil];
        return;
    }
    
    AVCaptureMetadataOutput *captureOutput = [[AVCaptureMetadataOutput alloc] init];
    [captureOutput setMetadataObjectsDelegate:self queue:_queue];
    self.captureOutput = captureOutput;
    
    self.captureSession = [[AVCaptureSession alloc] init];
    [self.captureSession addInput:captureInput];
    [self.captureSession addOutput:captureOutput];
    
    CGFloat w = 1920.f;
    CGFloat h = 1080.f;
    if ([self.captureSession canSetSessionPreset:AVCaptureSessionPreset1920x1080]) {
        self.captureSession.sessionPreset = AVCaptureSessionPreset1920x1080;
    } else if ([self.captureSession canSetSessionPreset:AVCaptureSessionPreset1280x720]) {
        self.captureSession.sessionPreset = AVCaptureSessionPreset1280x720;
        w = 1280.f;
        h = 720.f;
    } else if ([self.captureSession canSetSessionPreset:AVCaptureSessionPreset640x480]) {
        self.captureSession.sessionPreset = AVCaptureSessionPreset640x480;
        w = 960.f;
        h = 540.f;
    }
    captureOutput.metadataObjectTypes = [captureOutput availableMetadataObjectTypes];
    CGRect bounds = [[UIScreen mainScreen] bounds];
    
    if (!self.prevLayer) {
        self.prevLayer = [AVCaptureVideoPreviewLayer layerWithSession:self.captureSession];
    }
    self.prevLayer.frame = bounds;
    self.prevLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
    [self.view.layer insertSublayer:self.prevLayer atIndex:0];
    //下面代码主要用来设置扫描的聚焦范围，计算rectOfInterest
    CGFloat p1 = bounds.size.height/bounds.size.width;
    CGFloat p2 = w/h;
    
    CGRect cropRect = CGRectMake(CGRectGetMinX(_cropRect) - kSNReaderScanExpandWidth, CGRectGetMinY(_cropRect) - kSNReaderScanExpandHeight, CGRectGetWidth(_cropRect) + 2*kSNReaderScanExpandWidth, CGRectGetHeight(_cropRect) + 2*kSNReaderScanExpandHeight);
    
//    CGRect cropRect = _cropRect;
    if (fabs(p1 - p2) < 0.00001) {
        captureOutput.rectOfInterest = CGRectMake(cropRect.origin.y /bounds.size.height,                         cropRect.origin.x/bounds.size.width,
                                                  cropRect.size.height/bounds.size.height,
                                                  cropRect.size.width/bounds.size.width);
    } else if (p1 < p2) {
        //实际图像被截取一段高
        CGFloat fixHeight = bounds.size.width * w / h;
        CGFloat fixPadding = (fixHeight - bounds.size.height)/2;
        captureOutput.rectOfInterest = CGRectMake((cropRect.origin.y + fixPadding)/fixHeight,
                                                  cropRect.origin.x/bounds.size.width,
                                                  cropRect.size.height/fixHeight,
                                                  cropRect.size.width/bounds.size.width);
    } else {
        CGFloat fixWidth = bounds.size.height * h / w;
        CGFloat fixPadding = (fixWidth - bounds.size.width)/2;
        captureOutput.rectOfInterest = CGRectMake(cropRect.origin.y/bounds.size.height,
                                                  (cropRect.origin.x + fixPadding)/fixWidth,
                                                  cropRect.size.height/bounds.size.height,
                                                  cropRect.size.width/fixWidth);
    }
}
```

##### 识别二维码图片
识别二维码图片的功能，最初的方案是使用三方库`ZXing`来实现，因为`ZXing`有人在维护，但`ZXing`识别相册中的二维码图片或本地的图片时，有些图片根本就识别不出来，且耗时较长，所以改为使用`ZBar`。在网上找到一篇文章[再见ZXing 使用系统原生代码处理QRCode](http://adad184.com/2015/09/30/goodbye-zxing/),实测发现使用系统原生代码来识别二维码图片时，在，iphone4s，系统为iOS9的手机发现传回来的数组为空。代码如下：
```objc 
- (NSString *)decodeQRImageWith:(UIImage*)aImage {
    NSString *qrResult = nil; 
    //iOS8及以上可以使用系统自带的识别二维码图片接口，但此api有问题，在一些机型上detector为nil。 
    if (iOS8_OR_LATER) { 
          CIContext *context = [CIContext contextWithOptions:nil];
          CIDetector *detector = [CIDetector detectorOfType:CIDetectorTypeQRCode context:context options:@{CIDetectorAccuracy:CIDetectorAccuracyHigh}];
          CIImage *image = [CIImage imageWithCGImage:aImage.CGImage];
          NSArray *features = [detector featuresInImage:image];
          CIQRCodeFeature *feature = [features firstObject]; 
          qrResult = feature.messageString;
      } else {
          ZBarReaderController* read = [ZBarReaderController new];
          CGImageRef cgImageRef = aImage.CGImage;
          ZBarSymbol* symbol = nil;
          for(symbol in [read scanImage:cgImageRef]) break;
             qrResult = symbol.data ;
            return qrResult;
     }
 }
 ```

无图无真相：

![14567CBE-E1D2-4FA7-AFA3-8B2037171F38.jpg](http://upload-images.jianshu.io/upload_images/117999-5dae9fc15755140c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

detector的值为nil，也就是说 

```objc     
CIDetector *detector = [CIDetector detectorOfType:CIDetectorTypeQRCode context:context options:@{CIDetectorAccuracy:CIDetectorAccuracyHigh}];
```
CIDetector的初始化方法无效。推测是苹果API的问题。 
##### 生成二维码图片
在`iOS8`及以上版本使用苹果的`API`生成二维码图片，代码如下：
```objc
- (UIImage *)encodeQRImageWithContent:(NSString *)content size:(CGSize)size {
    UIImage *codeImage = nil;
    if (iOS8_OR_LATER) {
        NSData *stringData = [content dataUsingEncoding: NSUTF8StringEncoding]; 
        //生成
      CIFilter *qrFilter = [CIFilter filterWithName:@"CIQRCodeGenerator"];
      [qrFilter setValue:stringData forKey:@"inputMessage"];
      [qrFilter setValue:@"M" forKey:@"inputCorrectionLevel"];
      UIColor *onColor = [UIColor blackColor];
      UIColor *offColor = [UIColor whiteColor];
      //上色
      CIFilter *colorFilter = [CIFilter filterWithName:@"CIFalseColor"
                                         keysAndValues:
                               @"inputImage",qrFilter.outputImage,
                               @"inputColor0",[CIColor colorWithCGColor:onColor.CGColor],
                               @"inputColor1",[CIColor colorWithCGColor:offColor.CGColor],
                               nil];

      CIImage *qrImage = colorFilter.outputImage;
      CGImageRef cgImage = [[CIContext contextWithOptions:nil] createCGImage:qrImage fromRect:qrImage.extent];
      UIGraphicsBeginImageContext(size);
      CGContextRef context = UIGraphicsGetCurrentContext();
      CGContextSetInterpolationQuality(context, kCGInterpolationNone);
      CGContextScaleCTM(context, 1.0, -1.0);
      CGContextDrawImage(context, CGContextGetClipBoundingBox(context), cgImage);
      codeImage = UIGraphicsGetImageFromCurrentImageContext();
      UIGraphicsEndImageContext(); 
      CGImageRelease(cgImage);
      } else {
          codeImage = [QRCodeGenerator qrImageForString:content imageSize:size.width];
      }
      return codeImage;
  }
```
`iOS8`以下使用`libqrencode`库来生成二维码图片。

#### 代码完善
`2015年12月11日` 

`QA`测试发现，服务端生成的二维码，使用`ZBar`识别不出来，但将这张图片保存到相册，然后发送就可以识别出来。最初的想法是要服务端修改生成的二维码，但安卓能够识别出来，此路不通，那只有看ZBar的源码了。
```objc
- (id <NSFastEnumeration>) scanImage: (CGImageRef) image {
        timer_start;
        int nsyms = [self scanImage: image
                          withScaling: 0];
      //没有识别出来，判断CGImageRef对象的宽和高是否大于640，大于或等于的话进行缩放再进行扫描
        if(!nsyms &&
           CGImageGetWidth(image) >= 640 &&
           CGImageGetHeight(image) >= 640)
            // make one more attempt for close up, grainy images
            nsyms = [self scanImage: image
                          withScaling: .5];

        NSMutableArray *syms = nil;
        if(nsyms) {
            // quality/type filtering
            int max_quality = MIN_QUALITY;
            for(ZBarSymbol *sym in scanner.results) {
                zbar_symbol_type_t type = sym.type;
                int quality;
                if(type == ZBAR_QRCODE)
                    quality = INT_MAX;
                else
                    quality = sym.quality;

                if(quality < max_quality) {
                    zlog(@"    type=%d quality=%d < %d\n",
                         type, quality, max_quality);
                    continue;
                }

                if(max_quality < quality) {
                    max_quality = quality;
                    if(syms)
                        [syms removeAllObjects];
                }
                zlog(@"    type=%d quality=%d\n", type, quality);
                if(!syms)
                    syms = [NSMutableArray arrayWithCapacity: 1];

                [syms addObject: sym];
            }
        }

        zlog(@"read %d filtered symbols in %gs total\n",
              (!syms) ? 0 : [syms count], timer_elapsed(t_start, timer_now()));
        return(syms);
      }
      if(max_quality < quality) {
          max_quality = quality;
          if(syms)
              [syms removeAllObjects];
      }
      zlog(@"    type=%d quality=%d\n", type, quality);
      if(!syms)
          syms = [NSMutableArray arrayWithCapacity: 1];

      [syms addObject: sym];
      }
  }
  zlog(@"read %d filtered symbols in %gs total\n",
        (!syms) ? 0 : [syms count], timer_elapsed(t_start, timer_now()));
  return(syms);
}
```

在这里就产生了一个解决有些二维码图片识别不出来的解决思路：将传过来的`UIImage`的宽和高设置为640，识别不出来再进行缩放识别。修改`UIImage`的代码如下：
```objc
-(UIImage *)TransformtoSize:(CGSize)Newsize {
    // 创建一个bitmap的context
    UIGraphicsBeginImageContext(Newsize);
    // 绘制改变大小的图片
    [self drawInRect:CGRectMake(0, 0, Newsize.width, Newsize.height)];
    // 从当前context中创建一个改变大小后的图片
    UIImage *TransformedImg=UIGraphicsGetImageFromCurrentImageContext();
    // 使当前的context出堆栈
    UIGraphicsEndImageContext();
    // 返回新的改变大小后的图片
    return TransformedImg;
}
```
这样类似于将`ZXing`中的`tryHard`设置为`YES`。识别不出来的二维码图片就可以识别了。

`2016年5月20日`
`遗留的bug`: 点击进入扫一扫界面，退出，再进入，这样重复5次左右，扫一扫之前的界面的会出现卡顿。
原因：多次进入扫一扫界面，再退出，因此界面未被系统回收，captureSession对象一直在运行，会造成内存泄露，引起上一个界面卡顿。
解决方案：在视图将要消失的时候，确保captureSession对象停止运行。
```objc
- (void)viewWillDisappear:(BOOL)animated {
  [super viewWillDisappear:animated];
  if ([self.captureSession isRunning]) {
      [self.captureSession stopRunning];
  }
}
```
`2018年4月28日`
识别二维码图片优化

近期通过`bugly`收集卡顿问题发现，二维码组件在识别二维码图片时，会出现卡顿问题。为优化识别速度，采用了三种方案，并分别进行测试，并对测试数据进行分析，最终挑选出最优的方案。

> 任务A：使用系统提供的CoreImage的CIDetector接口去识别二维码图片，返回对应的字符串；
任务B：使用zbar中的方法去识别二维码图片，返回对应的字符串。
```objc
//任务A
+ (NSString *)useSystemMethodDecodeImage:(UIImage *)image {
    NSString *resultString = nil;
    CIDetector *detector = [CIDetector detectorOfType:CIDetectorTypeQRCode
                                              context:nil
                                              options:@{CIDetectorAccuracy:CIDetectorAccuracyHigh}];
    if (detector) {
        CIImage *ciImage = [CIImage imageWithCGImage:image.CGImage];
        NSArray *features = [detector featuresInImage:ciImage];
        CIQRCodeFeature *feature = [features firstObject];
        resultString = feature.messageString;
    }
    return resultString;
}
//任务B
+ (NSString *)useZbarMethodDecodeImage:(UIImage *)image {
    UIImage *decodeImage = image;
    if (decodeImage.size.width < 641) {
        decodeImage = [decodeImage TransformtoSize:CGSizeMake(640, 640)];
    }
    QRCodeZBarReaderController* read = [QRCodeZBarReaderController new];
    CGImageRef cgImageRef = decodeImage.CGImage;
    QRCodeZBarSymbol *symbol = nil;
    for(symbol in [read scanImage:cgImageRef]) break;
    return symbol.data;
}
```

- 方案A：先执行任务A，如果获取到的字符串为空，再执行任务B。

```objc 
+ (NSString *)planOneDecodeWithImage:(UIImage *)image index:(NSInteger)index{
 
    NSMutableString *costTimeInfo = [NSMutableString stringWithFormat:@"%ld\r\n",index];
    CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();
    NSString *detectorString = [MUIQRCodeDecoder useSystemMethodDecodeImage:image];
    CFAbsoluteTime detectorCostTime = (CFAbsoluteTimeGetCurrent() - startTime);
    
    [costTimeInfo appendString:[NSString stringWithFormat:@"detector : %f ms\r\n",detectorCostTime *1000.0]];
    
    NSAssert(detectorString.length > 0, @"detector fail!");
    CFAbsoluteTime zbarStartTime = CFAbsoluteTimeGetCurrent();
    NSString *zbarSymbolString = [MUIQRCodeDecoder useZbarMethodDecodeImage:image];
    NSAssert(zbarSymbolString.length > 0, @"zbar fail!");
    CFAbsoluteTime zbarCostTime = (CFAbsoluteTimeGetCurrent() - zbarStartTime);
    
    [costTimeInfo appendString:[NSString stringWithFormat:@"zbar : %f ms\r\n",zbarCostTime *1000.0]];
    
    CFAbsoluteTime totalCostTime = (CFAbsoluteTimeGetCurrent() - startTime);
    
    [costTimeInfo appendString:[NSString stringWithFormat:@"total cost : %f ms\r\n",totalCostTime *1000.0]];
    [costTimeInfo appendString:[NSString stringWithFormat:@"detectorString : %@ ms\r\n",detectorString]];
    [costTimeInfo appendString:[NSString stringWithFormat:@"zbarSymbolString : %@ ms\r\n",zbarSymbolString]];
    return [costTimeInfo copy];
}
```
- 方案B：同时执行任务A和任务B，两者均执行完后，返回识别的结果； 
```objc 
+ (NSString *)planTwoDecodeWithImage:(UIImage *)image index:(NSInteger)index { 
    __block NSMutableString *costTimeInfo = [NSMutableString stringWithFormat:@"%ld\r\n",index];
    __block NSString *detectorString = nil;
    __block NSString *zbarSymbolString = nil;
    
    CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();
    
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
        detectorString = [MUIQRCodeDecoder useSystemMethodDecodeImage:image];
        NSAssert(detectorString.length > 0, @"detector fail!");
        CFAbsoluteTime costTime = (CFAbsoluteTimeGetCurrent() - startTime);
        [costTimeInfo appendString:[NSString stringWithFormat:@"detector : %f ms\r\n",costTime *1000.0]];
    });
    
    dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
        zbarSymbolString = [MUIQRCodeDecoder useZbarMethodDecodeImage:image];
        NSAssert(zbarSymbolString.length > 0, @"zbar fail!");
        CFAbsoluteTime costTime = (CFAbsoluteTimeGetCurrent() - startTime);
        [costTimeInfo appendString:[NSString stringWithFormat:@"zbar : %f ms\r\n",costTime *1000.0]];
    });
    
    dispatch_group_notify(group, dispatch_get_global_queue(0,0), ^{
        dispatch_semaphore_signal(semaphore);
    });
    
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    
    CFAbsoluteTime totalCostTime = (CFAbsoluteTimeGetCurrent() - startTime);
    [costTimeInfo appendString:[NSString stringWithFormat:@"total cost : %f ms\r\n",totalCostTime *1000.0]];
    [costTimeInfo appendString:[NSString stringWithFormat:@"detectorString : %@ ms\r\n",detectorString]];
    [costTimeInfo appendString:[NSString stringWithFormat:@"zbarSymbolString : %@ ms\r\n",zbarSymbolString]];
    return [costTimeInfo copy];
}
```
- 方案C：同时执行任务A和任务B
1、任务A先执行完且识别成功，返回识别结果；
2、任务B先执行完且识别成功，返回识别结果；
3、任务A和任务B均识别失败，两者均执行完后，返回识别的结果。
```objc
+ (NSString *)planThreeDecodeWithImage:(UIImage *)image index:(NSInteger)index {
    __block NSMutableString *costTimeInfo = [NSMutableString stringWithFormat:@"%ld\r\n",index];
    __block NSString *detectorString = nil;
    __block NSString *zbarSymbolString = nil;
    __block BOOL isNeedSendSignal = YES;
    
    CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();
    
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
        detectorString = [MUIQRCodeDecoder useSystemMethodDecodeImage:image];
        //NSAssert(detectorString.length > 0, @"detector fail!");
        CFAbsoluteTime costTime = (CFAbsoluteTimeGetCurrent() - startTime);
        [costTimeInfo appendString:[NSString stringWithFormat:@"detector : %f ms\r\n",costTime *1000.0]];
        if (detectorString.length > 0 && isNeedSendSignal) {
            isNeedSendSignal = NO;
            dispatch_semaphore_signal(semaphore);
        }
    });
    
    dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
        zbarSymbolString = [MUIQRCodeDecoder useZbarMethodDecodeImage:image];
        //NSAssert(zbarSymbolString.length > 0, @"zbar fail!");
        CFAbsoluteTime costTime = (CFAbsoluteTimeGetCurrent() - startTime);
        [costTimeInfo appendString:[NSString stringWithFormat:@"zbar : %f ms\r\n",costTime *1000.0]];
        if (zbarSymbolString.length > 0 && isNeedSendSignal) {
            isNeedSendSignal = NO;
            dispatch_semaphore_signal(semaphore);
        }
    });
    
    dispatch_group_notify(group, dispatch_get_global_queue(0,0), ^{
        if (isNeedSendSignal) {
            dispatch_semaphore_signal(semaphore);
        }
    });
    
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    
    CFAbsoluteTime totalCostTime = (CFAbsoluteTimeGetCurrent() - startTime);
    [costTimeInfo appendString:[NSString stringWithFormat:@"total cost : %f ms\r\n",totalCostTime *1000.0]];
    [costTimeInfo appendString:[NSString stringWithFormat:@"detectorString : %@ ms\r\n",detectorString]];
    [costTimeInfo appendString:[NSString stringWithFormat:@"zbarSymbolString : %@ ms\r\n",zbarSymbolString]]; 
    return [costTimeInfo copy];
}
``` 
测试数据如下所示:(取了前10张图片）

![识别二维码图片耗时.png](https://upload-images.jianshu.io/upload_images/117999-04bca1bbc1f28805.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分析测试数据发现：
1、在测试第一张二维码图片时，总耗时均较大，如果第一次识别使用的是系统方法，耗时超过500ms，这也是为什么会出现卡顿的原因；
2、使用系统方法去识别二维码图片时，如果不是第一次去识别，耗时较小，在65ms以内；
3、使用zbar的方法去识别二维码图片，耗时均值在200ms以内；
4、在方案C中，如果第一次使用系统方法，耗时为226ms。

总结得出，从优化卡顿问题的角度出发，使用方案C最优，同时发现，如果使用系统方法能识别出二维码图片，在初始化之后（也就是第二次使用），耗时最短。同时因为在实际的使用场景中，图片是一张一张识别的，识别过程有一个间隔时间，如果已经使用系统方法识别过二维码图片，那下次识别就能达到最优。所以使用方案C的话，最差情况均值在200ms左右，最好的情况和方案A中第二次使用系统方法耗时基本一致。综合考虑，使用方案C。

#### 小结
在实际的项目开发过程中，设想的情况和实际情况会存在偏差，需要自己时刻使用性能调优工具，根据数据去进行优化，而不能想当然的认为某种方式是最优的。
源码和demo请点[这里](https://github.com/leverTsui/QRCodeDemo.git)
参考的文章链接如下
[再见ZXing 使用系统原生代码处理QRCode](http://adad184.com/2015/09/30/goodbye-zxing/)
[IOS二维码扫描,你需要注意的两件事](http://blog.cnbluebox.com/blog/2014/08/26/ioser-wei-ma-sao-miao/)
[[Zbar算法流程介绍](http://blog.csdn.net/u013738531/article/details/54574262)](http://blog.csdn.net/u013738531/article/details/54574262)