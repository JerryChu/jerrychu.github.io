---
title: Xcode展示自定义warning提示
date: 2018-08-05 16:56:47
tags:
- iOS
- Xcode
---

在 Xcode 的 _Build Setting_ 中，可以进行一些warning设置，便于在编译阶段提前发现代码问题，比如 _隐式类型转换_ 或者 _返回类型不匹配_ 等。但有时我们需要展示一些 __自定义的warning提示__ ，最常见就是对代码中的`TODO`、`FIXME`等标签进行warning提示，以免之后忘记修改。我们想要的效果是这个样子：

![img](https://wx3.sinaimg.cn/mw690/83e01499gy1ftywvsuc4zj21kw07qq4f.jpg)


### 如何在Xcode中展示warning提示

如何让 Xcode 展示出我们自定义的waring呢？其实很简单，只要我们按如下格式输出一段文本，Xcode就会把这段文本在对应的位置展示出来。

	/path/to/file:row: warning: text
	/path/to/file:row:column: warning: text

那么我们怎么去输出这段文本呢？我们知道Xcode的 _Build Phases_ 中，可以添加 _Run Script_ 。没错，我们只需要在 _Run Script_ 中输出这段文本就行了。  

![img](https://wx4.sinaimg.cn/mw690/83e01499gy1ftywvt6v6aj21900cgq59.jpg)

点击图中左上角的“+”，选择 `New Run Script Phase`，在输入框中添加如下脚本，然后编译。

    echo "${SRCROOT}/Demo/ViewController.m:1:1: warning: haha"	

编译之后，我们就能看到Xcode左侧的warning列表中出现了一条warning，点击这条warning，就能够像Xcode自己输出的warning一样，自动定位到对应的位置。这样我们就能够在`ViewController.m`的第一行第一列看到一条warning提示。

![img](https://wx2.sinaimg.cn/mw690/83e01499gy1ftywvs5efjj20g0046weu.jpg)

![img](https://wx3.sinaimg.cn/mw690/83e01499gy1ftywvso3xnj21j009swg0.jpg)

按同样的方法也可以展示自定义的error提示，格式如下：

	/path/to/file:row: error: text
	/path/to/file:row:column: error: text.
	

### 如何检测代码中的指定标签

知道了如何展示自定义的warning后，接下来的目标就很清晰了，即如何去找到`TODO`、`FIXME`等标签所在的文件及行数。  
为了实现这个功能，我们需要遍历每一个文件的每一行代码，检测代码中是否出现了对应的标签。这一步我们可以用下面的脚本实现。参考自 [https://krakendev.io/blog/generating-warnings-in-xcode](https://krakendev.io/blog/generating-warnings-in-xcode) 。

	# 需要检测的标签
	TAGS="TODO:|FIXME:"
	# 对目录下所有.m .h文件进行逐行的匹配检测
	find "${SRCROOT}" \( -name "*.m" -or -name "*.h" \) -print0 \  
   		 | xargs -0 egrep --with-filename --line-number --only-matching   "($TAGS).*\$"
    	 
如果有匹配行，那我们就需要按上面提到的指定格式进行输出。由于我们已经能拿到文件地址以及匹配行，所以接下来只需要按格式输出即可。这里使用perl语言可以很方便地实现。
    	 
    TAGS="TODO:|FIXME:"  
	 find "${SRCROOT}" \( -name "*.m" -or -name "*.h" \) -print0 \  
   		 | xargs -0 egrep --with-filename --line-number --only-matching   "($TAGS).*\$" \  
    	 | perl -p -e "s/($TAGS)/ warning: \$1/"
    	 
再次编译，左侧的warning列表中就会出现另一个warning，点击这个warning，会看到所在行有一个`TODO:`标签。这样就实现了添加自定义warning提示的功能。

![img](https://wx3.sinaimg.cn/mw690/83e01499gy1ftywvsuc4zj21kw07qq4f.jpg)

### 总结
了解Xcode展示warning的原理后，我们可以基于这个功能做很多有趣的事情。除了本文提到的对指定标签展示warning提示外，还可以在Xcode中集成重复代码检测工具，或者进行代码风格提示等。Xcode是一个强大的编译器，还有更多有意思的功能等着我们去探索。

示例项目地址：[https://github.com/JerryChu/Xcode-tips](https://github.com/JerryChu/Xcode-tips)。