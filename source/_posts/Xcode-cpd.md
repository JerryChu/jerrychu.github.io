---
title: Xcode重复代码检测
date: 2018-08-05 18:18:32
tags:
---

随着项目的推进和业务的增长，代码中不可避免地会出现很多重复代码，这些重复代码大多是从某个地方拷贝过去的，所以我们称之为`Copy-Paste-Code`。尤其是项目成员较多时，大家碰到不太熟悉的业务模块，经常会直接copy前人写好的代码，一来简单迅速，二来风险较低，但却会导致`Copy-Paste-Code`越来越多。这种代码让项目非常难以维护，一处修改，处处修改。  

### 重复代码检测工具

如何找出项目中的重复代码呢？现在有很多的重复代码检测工具（`Copy-Paste-Detector`，简称`CPD`）可以帮助我们完成这件事情。`CPD`的原理比较好理解，就是一个字符串匹配算法。如果两个代码片段完全相同，即认为这两处代码是`Copy-Paste-Code`。  

[PMD](https://pmd.github.io/)是一个比较流行的静态代码分析工具，支持多种语言。我们可以使用其提供的[CPD](https://pmd.github.io/pmd-6.6.0/pmd_userdocs_cpd.html)对OC代码做重复检测。  

#### 安装
使用`brew`就可以安装`PMD`

    brew install pmd

如果安装失败（像我一样...），也可以自行下载源码运行。mac上可以按如下命令执行

    $ cd $HOME
    $ curl -OL https://github.com/pmd/pmd/releases/download/pmd_releases%2F6.6.0/pmd-bin-6.6.0.zip
    $ unzip pmd-bin-6.6.0.zip
    $ alias pmd="$HOME/pmd-bin-6.6.0/bin/run.sh"

#### 检测
安装好`PMD`后，就可以直接检测项目中的重复代码。`PMD-CPD`有GUI工具，如下命令即可打开

    pmd cpdgui 

在GUI工具中选择目录、语言、和检测阈值（demo中使用的是20，实际项目中建议取值在100到150之前，视具体情况而定）后，点击 _Go_ 就可以检测了。

![img](https://wx4.sinaimg.cn/mw690/83e01499gy1ftz11juatyj21kk0uste5.jpg)

图中可以很清楚的看到哪些地方存在重复代码。有了这个检测结果，我们就可以拿来做针对性的修改与重构。  

### 集成到Xcode
但是，只使用GUI工具检测出来重复还是不能够避免大家写代码时随心所欲的 __Copy-Paste__, 为了进一步规避`Copy-Paste-Code`，我们决定将这个工具集成到Xcode中去，只要出现了`Copy-Paste-Code`，让Xcode在编译期就发出警告，督促大家放弃 __Copy-Paste__，花几分钟时间思考更为合理的解决方案。  

为了实现这个目的，我们需要分两步走：  
1. 找到所有重复代码及其所在的文件、行数  
2. 在Xcode中将warning展示出来

#### 生成检测结果
第一步使用`PMD-CPD`就可以实现，上面提到的`PMD-CPD`不仅有GUI工具，还支持直接将重复代码检测结果以指定格式输出到文件中。

    pmd cpd --files Demo --minimum-tokens 20 --language objectivec --encoding UTF-8 --format xml > output.xml

其中 _files_ 用于指定需要检测的文件列表，_minimum-tokens_ 用于设置最小重复阈值，_format_ 用于指定输出文件的格式。`PMD-CPD`支持xml、csv、txt等格式，为了方便下一步处理，这里使用xml格式。生成的xml文件结构如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<pmd-cpd>
    <duplication lines="7" tokens="23">
        <file line="24"
            path="..Demo/ViewController.m"/>
        <file line="36"
            path="..Demo/ViewController.m"/>
        <codefragment><![CDATA[- (int)hello0 {
    int a = 0, b = 0;
    int c = a + b;
    return c;
}

- (int)hello1 {]]></codefragment>
    </duplication>
</pmd-cpd>
```

#### 展示Xcode warnings

拿到这个文件后，我们就可以实施第二步了。关于如何实现在Xcode中生成自定义的warning提示，可以参考之前一篇文章[Xcode展示自定义warning提示](https://jerrychu.github.io/2018/08/05/Xcode-warnings/)。简而言之，我们需要在检测出来的每一处重复代码的位置，按如下形式在Xcode中输出一行warning：

    echo "${SRCROOT}/Demo/ViewController.m:1:1: warning: xxx copy-paste lines from xxx"    

在Xcode的 _Build Phases_ 中，我们增加一个新的 _Run Script_，并添加如下代码

    # 检测并输出结果到output.xml
    sh ./pmd_cpd/bin/run.sh cpd --files ${EXECUTABLE_NAME} --minimum-tokens 20 --language objectivec --encoding UTF-8 --format xml > output.xml
    # 解析xml并生成Xcode warning
    ruby parse.rb

解析脚本parse.rb代码如下：

```ruby
require 'rexml/document'
include REXML  

file = File.new('output.xml')
doc = Document.new(file)
root = doc.root
root.each_element('duplication') { |item| 
    duplicatedFiles = []
	item.each_element('file') { |file|
		duplicatedFiles.push(file)
	}
	item.each_element('file') { |file|
		duplicatedString = duplicatedFiles.select{|e| e != file}.map {|e| "#{e.attributes['path'].split('/').last}:#{e.attributes['line']}"}.join(', ')
		puts "#{file.attributes['path']}:#{file.attributes['line']}:1: warning: #{item.attributes['lines']} copy-pasted lines from: #{duplicatedString}"
	}	
}
```

添加后直接编译，我们就能在Xcode中的warning列表中发现如下warning，大功告成了！

![img](https://wx4.sinaimg.cn/mw690/83e01499gy1ftz1fm7w4zj20g40dkabd.jpg)

![img](https://wx4.sinaimg.cn/mw690/83e01499gy1ftz1h2nba0j21j20dgtaq.jpg)

### 总结
使用重复代码检测工具可以方便地检测出重复代码，进一步解析检测结果并以warning的形式集成到Xcode中，在编译期间就可以很清晰地看到代码中的warning，方便发现已有重复代码，同时避免产生新的重复代码。

示例项目地址：[https://github.com/JerryChu/Xcode-tips](https://github.com/JerryChu/Xcode-tips)。