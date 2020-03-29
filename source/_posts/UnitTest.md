---
title: iOS中的单元测试
date: 2020-03-29 14:25:47
tags:
- iOS
- UnitTest
- Xcode
---

单元测试是保证代码质量和代码正确性的重要手段。对于OC这类面向对象的编程语言来说，单元测试其实就是对方法的测试，我们通过单元测试来确保方法的行为符合预期。

示例项目地址：[https://github.com/JerryChu/UnitTestDemo](https://github.com/JerryChu/UnitTestDemo)。

### Xcode中编写单元测试

Xcode集成了`XCTest`单元测试框架，我们可以直接在Xcode中很方便的编写和运行单元测试。

#### 创建/添加单元测试Target

为了能在Xcode中运行单元测试，首先我们需要添加一个单元测试的Target。有两种方式添加单元测试Target:  

- 创建project时勾选“Include Unit Tests”，Xcode会自动创建一个单元测试Target

![勾选单元测试](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/includeUnitTest.png)

- 单独添加单元测试Target

![添加单元测试](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/newUnitTestTarget.png)

创建单元测试Target之后，Xcode中就会出现单元测试Target和对应的单元测试目录，我们在这个目录下添加单元测试类和单元测试方法，就可以运行单元测试了。

![单元测试Target](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/unitTestTarget.png)


#### 单元测试文件

添加单元测试Target之后，Xcode会自动创建一个单元测试文件，比如*DemoTests.m*。

```
@interface DemoTests : XCTestCase

@end

@implementation DemoTests

- (void)setUp {
    // Put setup code here. This method is called before the invocation of each test method in the class.
}

- (void)tearDown {
    // Put teardown code here. This method is called after the invocation of each test method in the class.
}

- (void)testExample {
    // This is an example of a functional test case.
    // Use XCTAssert and related functions to verify your tests produce the correct results.
}

- (void)testPerformanceExample {
    // This is an example of a performance test case.
    [self measureBlock:^{
        // Put the code you want to measure the time of here.
    }];
}
```

可以看到，单元测试类继承自`XCTestCase`，这是一个XCTest框架提供的单元测试基类，封装了一些单元测试通用的逻辑。  

*DemoTests*类的方法列表中，有`setUp`方法和`tearDown`两个方法，从命名和注释中我们可以看到，`setUp`方法是在每个单元测试方法运行开始之前调用一次，`tearDown`方法是在每个单元测试方法运行结束之后调用一次。因此，我们可以把初始化的逻辑写在`setUp`方法中，把清理重置等逻辑写在`tearDown`方法中。  

在`setUp`和`tearDown`方法下面就是单元测试方法了，其中*testExample*方法用于进行普通的单元测试，*testPerformanceExample*方法用于进行性能测试。无论是什么类型的测试，单元测试方法名都必须以**test**开头。如果一个方法的名字不是**test**开头的，那么Xcode就不会认为这是一个单元测试方法，自然就不会运行了。

![单元测试命名](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/name.png)


#### 新增单元测试

首先，我们要在单元测试Target下新增一个单元测试文件。

![添加单元测试](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/newFile.png)

![添加单元测试](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/addFile.png)

文件添加后，我们点击到文件中就可以看到和上述*DemoTests.m*一样的文件结构。我们在文件中添加单元测试方法，即可进行单元测试。

### Xcode中运行单元测试

在Xcode中有很多种方式可以运行单元测试。

1. 运行单个单元测试

![添加单元测试](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/runSingle.png)

输出结果为：

```
2020-03-29 15:07:45.858868+0800 Demo[43053:309552] 🎉 setUp
2020-03-29 15:07:45.859013+0800 Demo[43053:309552] 1⃣️ test case 1
2020-03-29 15:07:45.859128+0800 Demo[43053:309552] 🔚 tearDown
```

2. 运行单元测试类中的所有单元测试

![添加单元测试](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/runMulti.png)

输出结果为：

```
Test Case '-[DemoTests2 testAnother]' started.
2020-03-29 15:08:16.239300+0800 Demo[43156:310714] 🎉 setUp
2020-03-29 15:08:16.239469+0800 Demo[43156:310714] 2⃣️ test case 2
2020-03-29 15:08:16.239620+0800 Demo[43156:310714] 🔚 tearDown
Test Case '-[DemoTests2 testAnother]' passed (0.002 seconds).
Test Case '-[DemoTests2 testSomething]' started.
2020-03-29 15:08:16.240448+0800 Demo[43156:310714] 🎉 setUp
2020-03-29 15:08:16.240592+0800 Demo[43156:310714] 1⃣️ test case 1
2020-03-29 15:08:16.240738+0800 Demo[43156:310714] 🔚 tearDown
Test Case '-[DemoTests2 testSomething]' passed (0.001 seconds).
```

3. 运行单元测试Target中的所有单元测试

![添加单元测试](https://raw.githubusercontent.com/JerryChu/jerrychu.github.io/master/images/unittest/runAll.png)

点击后会运行单元测试Target中的所有文件的所有单元测试方法。

### 命令行中运行单元测试

XCTest框架编写的单元测试不仅可以在Xcode中运行，也可以直接在命令行中运行。

```
xcodebuild test -project Demo.xcodeproj -scheme Demo -destination 'platform=iOS Simulator,name=iPhone 11'
```

在命令行中执行单元测试之后，我们可以拿到单元测试执行的所有详细数据，包括单元测试总数、失败单元测试数、执行时长、代码覆盖率等等。接下来的文章，我们会继续详细介绍如何实现单元测试自动化运行和单元测试数据的提取等。

### 总结

Xcode中集成了单元测试框架XCTest，我们可以很方便地在Xcode中编写和运行单元测试。Xcode中的单元测试主要有以下特征：

- 单元测试类继承自XCTestCase类
- 单元测试方法以test开头
- 每个单元测试方法独立运行
- setUp和tearDown分别在单元测试方法执行开始之前和结束之后运行
- 运行单元测试默认需要启动模拟器/真机


示例项目地址：[https://github.com/JerryChu/UnitTestDemo](https://github.com/JerryChu/UnitTestDemo)。