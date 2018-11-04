---
title: iOS开发之桌面辅助开发工具
date: 2018-11-04 21:07:49
tags: 
- iOS
- Peertalk
- usbmux
---

在日常开发中，我们经常需要通过scheme打开指定的页面。这时候我们一般会在打开手机 _Safari_ ，在地址输入框中输入`myapp://somepage?pageid=xxx`进行页面的跳转。但是由于移动手机的尺寸限制，每次输入这些scheme都挺繁琐，效率低且容易出错。如果能直接在电脑上输入scheme，然后就能打开手机APP里的指定页面，那该多好啊。

> 模拟器可以直接通过 `xcrun simctl openurl booted scheme` 打开scheme  

考虑到需要在电脑上控制手机里的APP，那就肯定需要电脑和手机进行通信了，这里主要想到两种方式：  

- 使用WiFi连接，电脑和手机连接到同一个局域网中，使用socket通过网络协议通信。 
- 使用USB连接，将电脑和手机直接连接在一起。

WiFi连接的问题是连接不稳定，容易断开，我们从Xcode的无线调试功能中也能领略到。
USB连接的话，既然是有线连接，稳定性自然得到了保证，也不需要切换网络。但是如何通过USB来进行电脑和手机的通性呢？会不会开发起来很复杂？

进一步研究发现，Apple提供了一个叫 `usbmux`的东西来实现电脑和手机基于USB的通信，iTunes和Xcode都用到了这个东西，所以通过USB连接进行通信方案肯定是可行的了。  

接下来就是好好研究研究`usbmux`了，苹果的官方文档是这样说的：  

> During normal operations, iTunes communicates with the iPhone using something called “usbmux” – this is a system for multiplexing several “connections” over one USB pipe. Conceptually, it provides a TCP-like system – processes on the host machine open up connections to specific, numbered ports on the mobile device. (This resemblance is more than superficial – on the mobile device, usbmuxd actually makes TCP connections to localhost using the port number you give it.)

简单翻译一下，就是`usbmux`在USB协议上实现多路TCP连接，将USB通信抽象为TCP通信。这样一来，通过USB连接电脑和手机之后，电脑和手机就可以通过建立TCP连接进行通信了！这样说可能有些抽象，下面通过这个图来简要介绍下基于`usbmux`的电脑和手机通信流程。 

![img](https://wx1.sinaimg.cn/mw690/83e01499ly1fwwc0ffnz8j216u0ugdip.jpg)

1. 电脑端iTunes或其他第三方应用程序向`usbmuxd`发起连接，请求获取USB连接的通知。
2. 手机通过USB连接到电脑上
3. `usbmuxd`监听到有手机连接，并向手机发送一个packet
4. 手机回复
5. `usbmuxd`确认手机已经成功连接，并通知电脑端应用程序
6. 电脑端应用程序向`usbmuxd`发起建立新的一条连接，请求连接手机的指定端口
7. `usbmuxd`向手机发送“假的TCP”（基于USB的TCP） SYN packet
8. 手机收到SYN packet并在指定端口打开TCP连接
9. 手机回复 TCP SYN/ACK 确认端口已经打开可以进一步建立连接
10. `usbmuxd`向手机发送 ACK 完成握手
11. `usbmuxd`向电脑端应用程序发送连接建立成功的通知 
12. 之后电脑端应用程序向`usbmuxd`发送的socket，都会经过USB-TCP协议发送到手机的指定端口上，然后手机端的应用程序可以通过监听该端口接受信息。手机向电脑传递信息，也是一样的流程。

开源库[Peertalk](https://github.com/rsms/peertalk)就是封装了上面的一套流程，并提供了发起连接的接口以及收发消息的端口。这样一来，我们可以直接使用 _Peertalk_ 实现基于USB的电脑和手机通信了。  

下面以电脑端发送scheme信息给手机端应用程序为例，介绍下使用Peertalk进行通信的流程。

- 电脑端应用程序需要监听手机连接的通知

```
- (void)startListeningForDevices {
  NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];
  
  [nc addObserverForName:PTUSBDeviceDidAttachNotification object:PTUSBHub.sharedHub queue:nil usingBlock:^(NSNotification *note) {
  	// 手机插入处理
  	// ...
  	// 建立连接
	[self connectToUSBDevice];
  }];
  
  [nc addObserverForName:PTUSBDeviceDidDetachNotification object:PTUSBHub.sharedHub queue:nil usingBlock:^(NSNotification *note) {
  	// 手机拔出处理
  }];
}

- (void)connectToUSBDevice {
  PTChannel *channel = [PTChannel channelWithDelegate:self];
  [channel connectToPort:PTExampleProtocolIPv4PortNumber overUSBHub:PTUSBHub.sharedHub deviceID:connectingToDeviceID_ callback:^(NSError *error) {
  	// 连接建立成功
  }];
}

```

- 电脑端发出信息

```
- (IBAction)sendMessage:(id)sender {
    NSString *message = @"qqnews://detail?newsid=xxx&commentid=xxx";
    dispatch_data_t payload = PTExampleTextDispatchDataWithString(message);
    [channel sendFrameOfType:PTExampleFrameTypeTextMessage tag:PTFrameNoTag withPayload:payload callback:^(NSError *error) {
    	// 发送完成
    }];
  }
}

```

- 手机端监听该端口并处理接收到的消息

```
// 监听本地端口
  PTChannel *channel = [PTChannel channelWithDelegate:self];
  [channel listenOnPort:PTExampleProtocolIPv4PortNumber IPv4Address:INADDR_LOOPBACK callback:^(NSError *error) {
  }];

// 处理接受到的消息
- (void)ioFrameChannel:(PTChannel*)channel didReceiveFrameOfType:(uint32_t)type tag:(uint32_t)tag payload:(PTData*)payload {
  if (type == PTExampleFrameTypeTextMessage) {
    PTExampleTextFrame *textFrame = (PTExampleTextFrame*)payload.data;
    textFrame->length = ntohl(textFrame->length);
    NSString *scheme = [[NSString alloc] initWithBytes:textFrame->utf8text length:textFrame->length encoding:NSUTF8StringEncoding];
    // handle scheme
  }
}
```
这样就实现了在电脑端输入scheme打开手机APP指定页面的功能，再也不用在手机 _Safari_ 上输入一长串scheme了！但是这种方式有个缺陷，就是没法通过电脑端传递的scheme冷启动APP，因为接收端口消息并处理的逻辑都是在APP里，APP没启动，这些逻辑自然处理不了。这个问题接下来我会再研究研究。

话说回来，既然知道了如何让电脑和手机通信，那肯定不能仅满足于打开scheme这一个功能了！比如我们可以把客户端的日志通过这个流程发送给电脑，就可以在电脑上实时看客户端日志了。基于这个，我们可以做很多有意思的开发工具。    

另外Facebook开源的桌面开发工具Flipper也使用了Peertalk，提供了查看日志、页面视图等功能，同时支持iOS和android。但是作为一个开发辅助工具，Flipper搞得有点复杂了，各种依赖一大堆，不过其特性的实现代码和设计思想还是值得研究的。 

