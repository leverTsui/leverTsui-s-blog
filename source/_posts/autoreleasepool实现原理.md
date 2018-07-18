main.m中包含的代码如下所示,使用 clang -rewrite-objc 命令将下面的 Objective-C 代码重写成 C++ 代码：
```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSLog(@"Hello, World!");
    }
    return 0;
}
```
将会得到以下输出结果：
```objc
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);

struct __AtAutoreleasePool {
    __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
    ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
    void * atautoreleasepoolobj;
};

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_x__wh4dgr654mx__qc0b9bqyg5w0000gn_T_main_826771_mi_0);
    }
    return 0;
}
```
通过声明一个 `__AtAutoreleasePool` 类型的局部变量 `__autoreleasepool` 来实现 `@autoreleasepool {}` 。当声明 `__autoreleasepool` 变量时，构造函数 `__AtAutoreleasePool()` 被调用，即执行 `atautoreleasepoolobj = objc_autoreleasePoolPush()`; ；当出了当前作用域时，析构函数 `~__AtAutoreleasePool()` 被调用，即执行 `objc_autoreleasePoolPop(atautoreleasepoolobj)`; 。也就是说 `@autoreleasepool {}` 的实现代码可以进一步简化如下：

```objc 
/* @autoreleasepool */ {
    void *atautoreleasepoolobj = objc_autoreleasePoolPush();
    // 用户代码，所有接收到 autorelease 消息的对象会被添加到这个 autoreleasepool 中
    objc_autoreleasePoolPop(atautoreleasepoolobj);
}
```