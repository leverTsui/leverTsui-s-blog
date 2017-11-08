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
#### 小结
源码和demo请点[这里](https://github.com/hua16/QRCodeDemo.git)
参考的文章链接如下
[再见ZXing 使用系统原生代码处理QRCode](http://adad184.com/2015/09/30/goodbye-zxing/)
[IOS二维码扫描,你需要注意的两件事](http://blog.cnbluebox.com/blog/2014/08/26/ioser-wei-ma-sao-miao/)
[[Zbar算法流程介绍](http://blog.csdn.net/u013738531/article/details/54574262)](http://blog.csdn.net/u013738531/article/details/54574262)