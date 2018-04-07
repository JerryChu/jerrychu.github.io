---
title: 列表性能优化之cell数据缓存
date: 2018-03-25 18:53:02
tags:
- iOS
- MVVM
---

> 通过对列表cell的数据，如cell的高度、subView的布局数据等进行缓存，可以避免不必要的重复计算带来的性能开销，从而实现对列表性能的深度优化。同时可以结合MVVM中ViewModel的概念，进行cell数据的存储，使得代码结构更加清晰。

在客户端开发中，最经常使用的就是各种列表了，列表性能的好坏，很大程度上决定了一个应用的使用体验如何。关于如何优化列表性能，大家基本都能说出来一些基本方法，比如：

- cell复用
- 在子线程进行耗时操作，避免阻塞主线程
- 避免离屏渲染
- 图片预处理
- 减少subView数量
- 不要给cell动态添加subView
- ......

在实际开发过程中，我们经常会发现，即使已经采用了上面的方法，列表的性能还是不尽如人意。如何能更进一步地进行列表性能的优化呢？下面我们从列表中 __cell的数据缓存__ 方面探讨一下解决方案。

### cell高度缓存

对cell高度的缓存已经是业界比较通用的方案。列表每次展现cell时，都会执行回调方法获取cell的高度。

- 列表reload时，会重新计算所有cell的高度。
- 由于存在cell的复用，当从复用池中取出cell时，需要重新计算cell的高度。

正常情况下，每条数据对应的cell高度其实是一定的，当一条数据的对应的cell高度计算出来时，可以将高度`存到某个地方`，之后再展示这条数据时，就可以直接返回已经计算好的高度。关于cell高度的存储，一般有以下两种方式：

1. 存储在`Dictionary`中
这种方式需要每条数据都有一个唯一标识，作为存储高度的`key`

```objc
// self.cellHeightDict = @{"identifier_0" : "height_0", "identifier_1" : "height_1"};

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    // 获取数据的identifer
    if (self.cellHeightDict[identifier]) {
        return self.cellHeightDict[identifier].CGFloatValue;
    } else {
        // calculate height
        self.cellHeightDict[identifier] = @(height);
        return height;
    }
}
```

2. 存储在`Model`中
这种方式将数据计算出的高度作为一个属性，添加到数据对象中。相比于第一种方式，这种方式更易于理解，同时更加安全。

```objc
@interface Model : NSObject

@property(nonatomic) CGFloat calculatedHeight;
@property(nonatomic) BOOL isHeightCalculated;

@end

// ---

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    // 获取数据（model）
    if (model.isHeightCalculated) {
        return model.calculatedHeight;
    } else {
        // calculate height
        model.isHeightCalculated = YES;
        model.calculatedHeight = height;
        return height;
    }
}
```

cell高度的缓存很好地结局了高度多次重复计算带来的性能开销。沿着这种思路，我们可以发现需要被缓存的并不只是cell的高度。

### cell数据缓存
cell数据缓存方案不只局限于缓存cell的高度。cell及其subViews的布局数据，以及其他需要进行复杂计算的数据都可以缓存起来。
在日常开发过程中，我们经常遇到cell中有若干个垂直排列的label的情况，而每个label的高度都需要动态计算。一般来说，实现代码如下：

```objc
// cell中有两个label垂直排列的情况

// 计算高度时，需要计算每个label的高度
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    // 获取数据（model）
    if (model.isHeightCalculated) {
        return model.calculatedHeight;
    } else {
        CGSize size0 = [model.text0 sizeWithAttributes...];
        CGSize size1 = [model.text1 sizeWithAttributes...];
        CGFloat height = size0.height + size1.height;
        model.isHeightCalculated = YES;
        model.calculatedHeight = height;
        return height;
    }
}

// 展示时，对label进行布局
- (void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath {
    // ...

    CGSize size0 = [model.text0 sizeWithAttributes...];
    label0.text = model.text0;
    label0.frame = (CGRect){0, 0, size0.width, size0.height};

    CGSize size1 = [model.text1 sizeWithAttributes...];
    label1.text = model.text1;
    label1.frame = (CGRect){0, CGRectGetMaxY(label0.frame), size0.width, size0.height};

    // ...

}

```

对于cell的高度，由于已经进行了缓存，没有发生重复计算。但是对于lable的size，每次对cell进行布局时，都会重新计算一遍。实际上，这种的字符串的尺寸计算是非常消耗性能的。因此，这些数据也应该被缓存起来。
那么，这些数据该使用哪种方式缓存起来呢？

- 由于同一个identifier会对应多个数据（cell高度，label尺寸等），不适合直接使用词典存储。
- 把这些数据都作为model的属性，会导致model过于复杂。

这种问题很适合使用 [MVVM架构](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel) 来解决。


### MVVM
在`MVVM`中，`VM(ViewModel)`存储着`V(View)`布局所需要的数据。同样，对于每一个`Cell`（对应于`MVVM`中的`V(View)`），我们都可以创建一个对应的`ViewModel`来存储它布局所需要的数据。

每条数据（对应于`MVVM`中的`M(Model)`）同样需要一个唯一标识，用于存储其对应的`ViewModel`。基本结构如下：

```objc
// ViewModel
@interface ViewModel : NSObject

@property(nonatomic) CGFloat height; // cell高度
@property(nonatomic) CGFloat lable0Frame;
@property(nonatomic) CGFloat lable1Frame;

@end
```

按照这个思路，列表cell布局的基本的代码实现如下：

```objc
// self.viewModelDict = @{"identifier_0" : "viewModel_0", "identifier_1" : "viewModel_1"};

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    // 获取数据的identifer
    if (self.viewModelDict[identifier]) {
        ViewModel *vm = self.viewModelDict[identifier];
        return vm.heigth;
    } else {
        // calculate viewModel
        ViewModel *vm = [ViewModel new];
        // calculate cellHeight
        vm.height = height;
        // calculate label0's frame
        vm.label0Frame = label0Frame;
        // calculate label1's frame
        vm.label1Frame = label1Frame;
        self.viewModelDict[identifier] = vm;

        return height;
    }
}

- (void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath {
    // 获取数据的identifer
    ViewModel *vm = self.viewModelDict[identifier];

    label0.text = model.text0;
    label0.frame = vm.label0.frame;

    label1.text = model.text1;
    label1.frame = vm.label1.frame;

    // ...

}

```

当然，上面的ViewModel中并不仅局限于存储height、frame等数据，还可以存储许多布局时需要的数据，比如label对应的text或attributedText等。

总的来说，这种 __Cell数据缓存 + MVVM__ 的方式能够避免很多不必要的重复计算带来的性能开销，很好地提升列表的滚动流畅性；同时将计算布局的代码和实际布局UI的代码拆分开，代码结构更加清晰，并且为之后的进一步优化打好了基础（比如将布局代码放到子线程计算）。

<div id="container"></div>
<link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
<script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
<script>
  var gitment = new Gitment({
    id: 'location.href', // 可选。默认为 location.href
    owner: 'jerrychu', // 可以是你的GitHub用户名，也可以是github id
    repo: 'jerrychu.github.io',
    oauth: {
      client_id: '2820df553658e8bf2ed9',
      client_secret: 'c08951159957d795f7e1ec43cecc5b85fa954178',
    },
  })
  gitment.render('container')
</script>