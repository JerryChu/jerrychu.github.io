---
title: 谈谈单例设计模式
date: 2019-05-19 22:48:16
tags:
-  设计模式
- iOS
---

__单例模式__ 是最常被开发者使用的设计模式之一，同时，可能也是最常被 __滥用__ 的设计模式之一。  

关于单例模式的好处，网上有很多资料，这里就不再赘述了。本文以iOS 为例，探讨下单例可能带来哪些使用上的问题，以及如何避免这些问题。

### 问题

1. 不必要的提前初始化 

由于单例模式的特点，当我们访问单例对象的任何一个属性时，都会造成单例对象的初始化。很多时候，我们在调用单例对象的时候，并没有意识到这样调用带来的副作用，就导致了很多单例对象被提前初始化。  

假设我们有一个播放器单例类`CVideoPlayerManager `，这个单例类负责所有视频的播放。

```objc
// CVideoPlayerManager.h
@interface CVideoPlayerManager

@property(nonatomic, assign) BOOL enableRotation; // 是否允许播放器旋转 

- (void)playWithURL:(NSURL *)url;

@end

// CVideoPlayerManager.m
@implementation CVideoPlayerManager

+ (CVideoPlayerManager *)sharedInstance {                    
static dispatch_once_t once;                
static CVideoPlayerManager *manager;              
dispatch_once(&once, ^{                            
manager = [[CVideoPlayerManager alloc] init]; 
}); 
return manager;
}   

- (instancetype)init {
self = [super init];
if (self) {
// 播放器初始化
// 播放视图初始化
// 通知注册
// ...
}
return self;
}               

```
正常情况下，当我们第一次播放视频的时候，播放器才会被初始化。但是有时候我们在程序启动后，就想更新播放器的一些配置项，比如：

```objc
[CVideoPlayerManager sharedInstance].enableRotation = NO;
```

写完这句代码之后，播放器的配置就生效了，需求搞定！但是仔细看一下，这里调用了 `[CVideoPlayerManager sharedInstance]`方法，也就是说代码执行到这一行的时候，播放器实例就会被初始化了。而播放器的初始化代码里，往往包含着比较复杂的逻辑，这样的提前初始化，不仅仅导致了内存中过早的出现了一个播放器对象，更可能会造成代码逻辑上的问题。  
如果播放器单例初始化代码里还调用了一些其他单例类的方法，然后这些个单例类的初始化方法里又调用了再其他类的单例方法，整个应用程序内部的单例生命周期就会变的异常混乱。

2. 忘记销毁单例类

单例类使用起来很方便，我们也都知道单例类在内存中只会有一个对象（也可以创建多个对象，这里不展开说明了），但是我们经常会忘记单例对象有时也是需要 __销毁__ 的。很多情况下，我们创建的单例类并不需要存在于应用程序的整个生命周期。比如用户关闭某个模块后，这些模块特有的一些单例对象就不需要了，如果我们不把它们清理掉，它们就会一直保留在内存中。占用内存不说，还有可能执行一些不该执行的逻辑，带来安全隐患。

考虑有一个音频播放器单例，这个音频播放器有一个对应的播放控件，它需要随着页面的切换不断更新，重新attach到当前页面上；同时，当用户主动关闭音频播放控件时，音频停止播放同时把播放控件从页面上移除。  
由于这个播放控件需要出现在应用程序的各个页面上，我们把这个播放器设计成了一个单例类

```objc
// CAudioPlayerManager.h
@interface CAudioPlayerManager

+ (CAudioPlayerManager *)sharedInstance;
- (void)playWithURL:(NSURL *)url;

@end

// CAudioPlayerManager.m
@implementation CAudioPlayerManager

static CAudioPlayerManager *s_manager
+ (CAudioPlayerManager *)sharedInstance {                    
static dispatch_once_t once;                
dispatch_once(&once, ^{                            
s_manager = [[CAudioPlayerManager alloc] init]; 
}); 
return s_manager;
}   

- (instancetype)init {
self = [super init];
if (self) {
// 初始化
// 监听页面切换通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(_handlePageChanged:) name:CPageChangedNotification];
}
return self;
}               

- (void)_handlePageChanged:(NSNotification *)notificaion {
NSLog(@"page changed.");
[self     reAttach];
}

- (void)reAattach {
// 将播放条重新添加到当前页面上
}

```

我们看到，`CAudioPlayerManager`是一个单例类，它监听了当前页面变化的通知，并在收到通知后有一些处理逻辑。当需要播放音频时，我们创建一个该单例的对象，然后去执行相关的逻辑，这都没问题。  
但是，当用户关闭这个音频播放控件时，整个音频播放器就在页面上不存在了。而`CAudioPlayerManager`里并没有提供一个销毁单例的方法。也就是说，虽然我们把播放器关闭了，也把音频播放控件从页面上移除了，但是`CAudioPlayerManager`的这个单例还一直存在于内存中。它还会继续监听页面变化的通知，继续执行处理逻辑。如果 `reAattach` 方法里没有进行完备的状态检验，这时候的代码可能就出问题了。再者，退一步说，即使不会对代码逻辑造成影响，`_handlePageChanged:`方法里的 `NSLog` 也会一直输出，污染了设备日志。


### 解决方案

说了这么多，那该如何去避免这些问题呢？这就需要我们合理地使用单例，不要不分场景地滥用单例。

1. 这个类是否有必要作为单例类。

很多时候，我们都并不是需要一个单例类，而只是为了图方便，所以把它直接设计成一个单例类。这就需要我们在写代码之前，就想清楚自己是不是必须要使用单例模式，详细考虑单例模式的优缺点以及可能遇到的问题，比如创建实例的开销如何，是否有很多全局状态需要保存等。   
比如上面提到的视频播放器类 `CVideoPlayerManager`，其实不一定非得是一个单例类。我们只需要每次在播放视频时，重新创建一个播放器实例即可，这样也能避免很多全局状态带来的逻辑问题。当然一些全局的配置可以单独放到一个单例类里去，做统一的管理。


2. 即使有些全局状态需要保存，也不一定非得用单例类来实现。

我们在使用单例类时，经常是为了保存一些全局状态。但是即使是要保存全局状态，也不一定非要把这个类作为单例类使用。我们可以使用`static`变量来维护一个全局对象。  
下面提供一个简单的代码示例来说明如何使用static变量而不是单例类来维护全局对象。这里并不是说就不让用单例，只是提供另外一个可行的方案。大家在写代码时，还是需要自己去思考用那种实现方式比较合理。

```
// CDataUtils.h
@interface CDataUtils

+ (NSString *)urlAtIndex:(NSInteget)index;

@end


// CDataUtils.m
static NSArray<NSString *> *s_urlList = nil;

@implementation CDataUtils

+ (void)initialize {
static dispatch_once_t once;
dispatch_once(&once, ^{
s_urlList = ...;
});
}

+ (NSString *)urlAtIndex:(NSInteget)index {
return [urlList objectAtIndex:index];
}

@end

```

这样还有一个好处是，当我们需要给`CDataUtils`添加更多的类方法时，不会带来任何副作用。因为调用类方法不会涉及到任何实例的初始化。

3. 为单例提供一个销毁实例的方法

如果是像上面说的`CAudioPlayerManager`这种需要销毁当前实例的单例类，我们可以给单例类提供一个销毁实例的方法。这样当我们不想要当前的实例时，就可以直接调用这个方法把实例从内存中移除。  
同时，我们还需要提供一个判断当前是否有实例的方法，如果当前没有实例，同时也不需要去创建一个新的实例，那么就可以直接把调用逻辑return调，防止错误地创建了新的实例。

```objc
// CAudioPlayerManager.h
@interface CAudioPlayerManager

+ (CAudioPlayerManager *)sharedInstance;
+ (void)dispose;
+ (BOOL)isSingletonDisposed;

@end

// CAudioPlayerManager.m
@implementation CAudioPlayerManager

static CAudioPlayerManager *s_manager;              
+ (CAudioPlayerManager *)sharedInstance {                    
static dispatch_once_t once;                
dispatch_once(&once, ^{                            
s_manager = [[CAudioPlayerManager alloc] init]; 
}); 
return s_manager;
}   

+ (void)dispose {                                      
if (s_manager) {                            
s_manager = nil;
}                           
}

+ (BOOL)isSingletonDisposed {
return s_manager == nil;
}
```

### 总结

单例模式是个非常好用也非常实用的设计模式，但是任何设计模式，一旦滥用，都会带来很多问题，单例模式更是这样。合理使用单例模式，需要我们在写下决定使用单例之前，想清楚是否该使用单例模式，是否有其他更好的替代方案。
