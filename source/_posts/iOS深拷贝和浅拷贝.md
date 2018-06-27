title: iOS深拷贝和浅拷贝
tags:
  - Object-C
categories:
  - 基础知识
author: leveltsui
date: 2017-12-14 12:26:00
---
在工作中，有时会涉及到深拷贝和浅拷贝的内容，发现有些地方理解的不够透彻，所以在网上搜集资料总结一下，主要分四个方面来介绍下iOS中深拷贝和浅拷贝：
- 对象的拷贝；
- 集合的拷贝；
- 如何对集合进行深拷贝？
- 总结
#### 对象的拷贝
对对象进行拷贝，通过调用`copy`或`mutableCopy`方法来实现：
- 调用 `copy`方法返回的对象为不可变对象,需要实现`NSCopying`协议 ；
- 调用`mutableCopy`方法返回的对象为可变对象，需要实现`NSMutableCopying`协议 ；

![object copying](http://upload-images.jianshu.io/upload_images/117999-adeebf183df99baa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图是苹果文档中关于对象拷贝的实例图，从图中可知：
> 浅拷贝：`object A`和`object B`及其属性使用同样的内存地址，简单说明的话，可以认为是指针复制；
深拷贝：`object A`和`object B`及其属性使用不同的内存地址，简单说明的话，可以认为是内容复制。

下面通过分析`NSString`、`NSMutableString`、和自定义对象`DBTestModel`，调用`copy`和`mutableCopy`之后，分析其返回的对象及此对象的属性的内存地址来判断其行为是深拷贝还是浅拷贝。

##### NSString

![NSString.png](http://upload-images.jianshu.io/upload_images/117999-bcac53e449d8b0a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过打印的信息可知：
- 对象`string`调用`copy`方法后返回的对象`copySting`，其所对应的内存地址和对象`string`一致，即为指针复制；
- 对象`string`调用`mutableCopy`方法后返回的对象`mutaCopySting`，其所对应的内存地址和对象`string`不一致，即为指内容复制；

##### NSMutableString

![NSMutableString.png](http://upload-images.jianshu.io/upload_images/117999-0eab0dce73bccbec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过打印的信息可知：
- 对象`mutaString`调用`copy`方法后返回的对象`copyMutaString`，其所对应的内存地址和对象`mutaString`不一致，即为内容复制；
- 对象`mutaString`调用`mutableCopy`方法后返回的对象`mutaCopyMutaString`，其所对应的内存地址和对象`mutaString`不一致，即为指内容复制；

##### DBTestModel
下面为自定义的对象'DBTestModel': 

```objectivec
#import <Foundation/Foundation.h>
#import <Mantle/Mantle.h>

@interface DBTestModel : MTLModel

@property (nonatomic, copy) NSString *text;

@property (nonatomic, strong) NSArray *sourceArray;

@property (nonatomic, strong) NSMutableArray *mutableArray;

- (instancetype)initWithText:(NSString *)text
                 sourceArray:(NSArray *)sourceArray
                mutableArray:(NSMutableArray *)mutableArray;
@end
```
定义了三个属性，`text`、`sourceArray`、`mutableArray`。
```objectivec
- (void)testCustomObject {
    NSMutableArray *mutableArray = [NSMutableArray array];
    DBTestModel *model = [[DBTestModel alloc] initWithText:@"text"
                                               sourceArray:@[@"test1",@"test2"]
                                              mutableArray:mutableArray];
    DBTestModel *copyModel = [model copy];
    DBTestModel *mutaCopyModel = [model mutableCopy];
    
    NSLog(@"DBTestModel memory address");
    NSLog(@"original   :%p", model);
    NSLog(@"copy       :%p", copyModel);
    NSLog(@"mutableCopy:%p", mutaCopyModel);
    NSLog(@"\n");
    
    NSLog(@"text memory address");
    NSLog(@"original   :%p", model.text);
    NSLog(@"copy       :%p", copyModel.text);
    NSLog(@"mutableCopy:%p", mutaCopyModel.text);
    NSLog(@"\n");
    
    NSLog(@"sourceArray memory address");
    NSLog(@"original   :%p", model.sourceArray);
    NSLog(@"copy       :%p", copyModel.sourceArray);
    NSLog(@"mutableCopy:%p", mutaCopyModel.sourceArray);
    NSLog(@"\n");
    
    NSLog(@"mutableArray memory address");
    NSLog(@"original   :%p", model.mutableArray);
    NSLog(@"copy       :%p", copyModel.mutableArray);
    NSLog(@"mutableCopy:%p", mutaCopyModel.mutableArray);
    NSLog(@"\n"); 
}
```
打印结果如下：
![image.png](http://upload-images.jianshu.io/upload_images/117999-7edadb3411c425dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
分析其打印数据可知：
  
- 除`DBTestModel`实例对象中的属性`text`和`sourceArray`调用`copy`后，没有产生一个新的对象，为指针复制，其余均为内容复制。 

#### 集合的拷贝
![image.png](http://upload-images.jianshu.io/upload_images/117999-c329ce5e7a3eb4e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 不可变集合`NSArray`
```objectivec
- (void)testCollectiveCopy {
    NSMutableArray *mutableArray1 = [NSMutableArray array];
    DBTestModel *model1 = [[DBTestModel alloc] initWithText:@"text"
                                               sourceArray:@[@"test1",@"test2"]
                                              mutableArray:mutableArray1];
    
    NSMutableArray *mutableArray2 = [NSMutableArray array];
    DBTestModel *model2 = [[DBTestModel alloc] initWithText:@"text"
                                                sourceArray:@[@"test1",@"test2"]
                                               mutableArray:mutableArray2];
    
    NSMutableArray *mutableArray3 = [NSMutableArray array];
    DBTestModel *model3 = [[DBTestModel alloc] initWithText:@"text"
                                                sourceArray:@[@"test1",@"test2"]
                                               mutableArray:mutableArray3];
    NSArray *array = @[model1,model2,model3];
    NSArray *copyArray = [array copy];
    NSMutableArray *mutaCopyArray = [array mutableCopy];

    NSLog(@"\nNSArray memory address\noriginal   :%p\ncopy       :%p\nmutableCopy:%p\n",
          array,copyArray,mutaCopyArray);
 
    DBTestModel *firstCopyModel = [copyArray firstObject];
    DBTestModel *firstMutableCopyModel = [mutaCopyArray firstObject];
    NSLog(@"\nDBTestModel memory address\noriginal   :%p\ncopy       :%p\nmutableCopy:%p\n",
          model1,firstCopyModel,firstMutableCopyModel);
 
    NSLog(@"\nDBTestModel sourceArray memory address\noriginal   :%p\ncopy       :%p\nmutableCopy:%p\n",
          model1.sourceArray,firstCopyModel.sourceArray,firstMutableCopyModel.sourceArray); 
}
```
打印结果如下： 

![NSArray.png](http://upload-images.jianshu.io/upload_images/117999-b541a083c1018ea6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分析其打印数据可知：
- 除`NSArray`实例对象调用`mutableCopy`方法为内容复制外，其余的均为指针拷贝。

##### 可变集合`NSMutableArray`
测试代码如下：
```objectivec
- (void)testCollectiveMutacopy {
    NSMutableArray *mutableArray1 = [NSMutableArray array];
    DBTestModel *model1 = [[DBTestModel alloc] initWithText:@"text"
                                                sourceArray:@[@"test1",@"test2"]
                                               mutableArray:mutableArray1];
    
    NSMutableArray *mutableArray2 = [NSMutableArray array];
    DBTestModel *model2 = [[DBTestModel alloc] initWithText:@"text"
                                                sourceArray:@[@"test1",@"test2"]
                                               mutableArray:mutableArray2];
    
    NSMutableArray *mutableArray3 = [NSMutableArray array];
    DBTestModel *model3 = [[DBTestModel alloc] initWithText:@"text"
                                                sourceArray:@[@"test1",@"test2"]
                                               mutableArray:mutableArray3];
    
    NSArray *array1 = @[model1,model2,model3];
    NSMutableArray *mutaArray = [NSMutableArray arrayWithArray:array1];
    NSMutableArray *copyMutaArray = [mutaArray copy];
    NSMutableArray *mutaCopyMutaArray= [mutaArray mutableCopy];
    
    NSLog(@"\nNSMutableArray memory address\noriginal   :%p\ncopy       :%p\nmutableCopy:%p\n",
          mutaArray,copyMutaArray,mutaCopyMutaArray);
    
    DBTestModel *firstCopyModel = [copyMutaArray firstObject];
    DBTestModel *firstMutableCopyModel = [mutaCopyMutaArray firstObject];
    NSLog(@"\nDBTestModel memory address\noriginal   :%p\ncopy       :%p\nmutableCopy:%p\n",
          model1,firstCopyModel,firstMutableCopyModel);
    
    NSLog(@"\nDBTestModel sourceArray memory address\noriginal   :%p\ncopy       :%p\nmutableCopy:%p\n",
          model1.sourceArray,firstCopyModel.sourceArray,firstMutableCopyModel.sourceArray);
}
```
测试结果如下： 

![NSMutableArray.png](http://upload-images.jianshu.io/upload_images/117999-50c419ac45f0dd51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分析其打印数据可知： 
除`NSMutableArray `实例对象调用`copy`和`mutableCopy`方法为内容复制外，数组内容的元素均为指针拷贝。
结合上述测试代码得出的测试数据，得出如下的表格： 
![image.png](http://upload-images.jianshu.io/upload_images/117999-cc6317d514f6c864.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上图可以看出，`NSArray`或`NSMutableArray`对象调用`copy`或`mutableCopy`时，得到的集合中的元素均为指针拷贝，如果想要实现集合对象的深拷贝，应该怎么办呢？

- #### 如何对集合进行深拷贝？
- ##### 集合的单层深复制 (one-level-deep copy)
可以用 `initWithArray:copyItems:` 将第二个参数设置为YES即可深复制，如
```
 NSArray *copyArray = [[NSArray alloc] initWithArray:array copyItems:YES];
```
如果你用这种方法深复制，集合里的每个对象都会收到 copyWithZone: 消息。同时集合里的对象需遵循 NSCopying 协议，否则会崩溃。
![image.png](http://upload-images.jianshu.io/upload_images/117999-8d496f137b153a4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

得到的结果如下：

![image.png](http://upload-images.jianshu.io/upload_images/117999-a48a67077a1f4d4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
从打印结果可以看出，拷贝后和拷贝前的数组中的`DBTestModel`对象的`sourceArray`的内存地址是相同的，这种拷贝方式只能够提供一层内存拷贝(`one-level-deep copy`)，而非真正的深复制。

- ##### A true deep copy
使用归档来实现:

```objectivec
NSArray *copyArray = [NSKeyedUnarchiver unarchiveObjectWithData:
       [NSKeyedArchiver archivedDataWithRootObject:array]];
``` 

####小结
在项目中遇到需要需要对对象进行拷贝时，需要理解以下两点：
1、对系统自带的不可变对象进行`copy`时，为指针复制；
2、对集合类对象进行`copy`或`mutableCopy`时，集合类的元素均为指针复制；
3、如只需对集合进行单层深拷贝，可以使用`initWithArray:copyItems:`类似的方法，将第二个参数设为`YES`来实现，如需实现集合的完全复制，可以使用归解档来实现；
4、第三方库` Mantle`中的`MTLModel`类有实现`NSCoding`和`NSCopying`协议，自定义的类继承`MTLModel`类即可实现`NSCoding`和`NSCopying`协议。

#### 参考链接
[Object copying](https://developer.apple.com/library/content/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectCopying.html)
[Copying Collections](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Articles/Copying.html)
 