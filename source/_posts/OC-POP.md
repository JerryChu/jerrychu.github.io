---
title: 面向协议的第三方分享模块实现（OC）
date: 2018-04-07 16:40:44
tags:
- iOS
- POP
---

第三方分享模块是各种各样的APP中必不可少的一个模块，最常用的就是分享到微信和QQ。本文基于`面向协议编程`（Protocol Oriented Programming，POP）的思想，介绍如何实现一个 __简单灵活__、__易于扩展__ 的第三方分享模块。

通常，我们需要实现以下几种功能：

- 分享一段文本
- 分享一张图片
- 分享一个网页
- ......

文本分享和图片分享比较简单，需要的数据较少。下面以微信的网页分享为例，介绍如何使用面向协议的思想实现分享功能。

网页分享，也就是分享了一个网页，用户在微信或QQ中点击分享card（如下图所示），会跳转到一个网页去。通常，这个分享card需要展示内容标题、内容描述、缩略图等数据，因此进行网页分享时，一般需要传递以下数据：

![img](https://wx2.sinaimg.cn/mw690/83e01499gy1fq45p5hwlxj20mc08zmyo.jpg)

- 分享页面的url
- 分享card的标题
- 分享card的描述
- 分享card的缩略图

在微信分享SDK中，需要使用到以下两个对象：

```objc
@interface WXMediaMessage : NSObject
/** 标题
* @note 长度不能超过512字节
*/
@property (nonatomic, retain) NSString *title;
/** 描述内容
* @note 长度不能超过1K
*/
@property (nonatomic, retain) NSString *description;
/** 缩略图数据
* @note 大小不能超过32K
*/
@property (nonatomic, retain) NSData   *thumbData;
/**
* 多媒体数据对象，可以为WXImageObject，WXMusicObject，WXVideoObject，WXWebpageObject等。
*/
@property (nonatomic, retain) id        mediaObject;
// ...
@end
```

```objc
@interface WXWebpageObject : NSObject
/** 网页的url地址
* @note 不能为空且长度不能超过10K
*/
@property (nonatomic, retain) NSString *webpageUrl;
// ...
@end
```

按照`面向对象编程`的思想，假设要分享的数据是一个`ShareModel`，你很快就实现了这个功能：

```objc
@implementation ShareHelper
+ (void)shareObjectToWeixin:(ShareModel *)model {
	WXWebpageObject *object = [WXWebpageObject object];
	object.webpageUrl = model.url;
	WXMediaMessage *message = [WXMediaMessage message];
	message.title = model.title;
	message.description = model.description;
	message.thumbData = model.thumbnailData;
	message.mediaObject = object;
	
	SendMessageToWXReq *req = [[SendMessageToWXReq alloc] init];
	req.message = message;
	[WXApi sendReq:req];
}
@end
```

代码并不复杂，用着也方便。有一天，产品过来和你说另一个页面也要支持分享。你看到另一个页面分享时用到的对象是`AnotherObject`，于是你马上`Ctrl+C`、`Ctrl+V`，不到一分钟就搞定了这个需求。

```objc
@implementation ShareHelper
+ (void)shareAnotherObjectToWeixin:(ShareModel *)model {
	WXWebpageObject *object = [WXWebpageObject object];
	object.webpageUrl = model.url;
	WXMediaMessage *message = [WXMediaMessage message];
	message.title = model.title;
	message.description = model.description;
	message.thumbData = model.thumbnailData;
	message.mediaObject = object;
	
	SendMessageToWXReq *req = [[SendMessageToWXReq alloc] init];
	req.message = message;
	[WXApi sendReq:req];
}
@end
```
正当你沉浸在实现需求的喜悦中时，产品又跑过来说，你这个分享功能还需要支持使用动态下发的数据进行分享，你发现需要支持分享词典数据。虽然有些纠结，你还是在`ShareHelper`中又加了一个方法：

```objc
@implementation ShareHelper
+ (void)shareDictionaryToWeixin:(NSDictionay *)dict {
	WXWebpageObject *object = [WXWebpageObject object];
	object.webpageUrl = dict[@"url"];
	WXMediaMessage *message = [WXMediaMessage message];
	message.title = dict[@"title"];
	message.description = dict[@"description"];
	message.thumbData = dict[@"thumbnailData"];
	message.mediaObject = object;
	
	SendMessageToWXReq *req = [[SendMessageToWXReq alloc] init];
	req.message = message;
	[WXApi sendReq:req];
}
@end
```

实现功能之后，看着`ShareHelper.h`，作为一个优秀的程序员，你有点方，因为你感觉到需要提供的接口会越来越多。

```objc
@interface ShareHelper : NSObject
// 分享ShareModel数据
+ (void)shareObjectToWeixin:(ShareModel *)model;
// 分享AnotherShareModel数据
+ (void)shareAnotherObjectToWeixin:(ShareModel *)model;
// 分享Dictionary数据
+ (void)shareDictionaryToWeixin:(NSDictionay *)dict;
@end
```

那么涉及到多种不同数据的分享，有什么比较好的办法能让 `ShareHelper` 别再继续膨胀下去呢？
使用`面向协议编程`的思想来分析这个问题，就会豁然开朗了。
从分享模块本身来说，其实它并不需要关注外部传递的是`ShareModel`、`AnotherShareModel`还是`NSDcitionary`，它只需要 __网页url__ 、 __标题__ 、 __简介__ 、 __缩略图__ 这四个数据。因此，无论是什么对象，只要能提供这四个数据，分享模块就可以把它分享出去。

```objc
@protocol Shareable <NSObject>

- (NSString *)shareTitle;
- (NSString *)shareDescription;
- (NSString *)shareUrl;
- (NSData *)shareThumbnailData;

@end
```

定义好这个协议之后，`ShareHelper`会变得非常简单：

```objc
@interface ShareHelper : NSObject
+ (void)shareToWeixin:(id<Shareable>)shareableModel;
@end

@implementation ShareHelper
+ (void)shareToWeixin:(id<Shareable>)shareableModel {
	WXWebpageObject *object = [WXWebpageObject object];
	object.webpageUrl = [shareableModel shareUrl];
	WXMediaMessage *message = [WXMediaMessage message];
	message.title = [shareableModel shareTitle];
	message.description = [shareableModel shareDescription];
	message.thumbData = [shareableModel shareThumbnailData];
	message.mediaObject = object;
	
	SendMessageToWXReq *req = [[SendMessageToWXReq alloc] init];
	req.message = message;
	[WXApi sendReq:req];
}
@end
```

这时，你就可以自豪地和同事们说，以后你们不论想分享啥，只要实现`Shareable`这个协议，就能给你分享出去！

对于文本分享和图片分享，也只需要提供对应的协议即可，实现了该协议的数据，就可以按照对应的形式分享出去。

```objc
@protocol ShareableText <NSObject>
- (NSString *)shareText;
@end

@protocol ShareableImage <NSObject>
- (UIImage *)shareImage;
@end
```

```objc
@interface ShareHelper : NSObject
+ (void)shareTextToWeixin:(id<ShareableText>)shareableModel;
+ (void)shareImageToWeixin:(id<ShareableImage>)shareableModel;
@end
```

习惯了`面向对象编程`之后，我们经常会发现写出的代码不够通用、不好扩展，尤其是对于工具类的方法。如果能将`面向对象编程`和`面向协议编程`结合起来使用，扬长避短，经常可以达到事半功倍的效果。
