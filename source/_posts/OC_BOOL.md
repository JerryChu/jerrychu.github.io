---
title: 探索OC中的BOOL类型
date: 2018-01-21 20:12:20
tags:
---

Obejctive-C 中的 `BOOL` 类型定义在 `<objc.h>`

	#if (TARGET_OS_IPHONE && __LP64__) || TARGET_OS_WATCH   
	#define OBJC_BOOL_IS_BOOL 1 
	typedef bool BOOL; 
	#else 
	#define OBJC_BOOL_IS_CHAR 1 
	typedef signed char BOOL; 
	#endif 

可见，在32位（准确的说，应该是非64位）机器上，OC中的 `BOOL` 其实并不是我们熟悉的C语言中的 `bool` ，而是 `signed char` 类型，所以OC中的 `BOOL` 所能存储的数值不止是 0 和 1 ，是 -128~127。

> C99提供了\_Bool类型，\_Bool依然仍是整数类型，但只能赋值为 0 或 1，非 0 值都会被存储为 1 。  
> C99还提供了<stdbool.h>，其中定义了 bool 代表\_Bool，并且定义了 true 和 false，true 代表 1，false 代表 0。

（所以发明 Objective－C 语言那会儿，C也还没有 bool 类型呢）  
同时，OC 中还有 `YES` 和 `NO` 两个宏，也是在 `<objc.h>` 中定义，其中 `YES` 代表 1，`NO` 代表 0。

	#if __has_feature(objc_bool)
	#define YES __objc_yes
	#define NO  __objc_no
	#else
	#define YES ((BOOL)1)
	#define NO  ((BOOL)0)
	#endif 

那么考虑下面这种情况，某个函数返回一个 `int` 值，在 `check` 函数中通过判断这个函数的返回值来进行之后的操作，那么在32位的机器中运行，`check` 函数会打印出 `YES` 还是 `NO` 呢？

	- (int)returnValue {
    	// use 256 as an example
    	return 256;
	}

	- (void)check {
		BOOL yesOrNo = (BOOL)[self returnValue];
    	if (yesOrNo) {
        	NSLog(@"YES");
    	} else {
       		NSLog(@"NO");
    	}
	}
	
可能大家觉得想都不用想，肯定是输出 `YES`。那么实际结果可能会让你恨郁闷，因为输出的是 `NO`。到底是为什么呢？ 

还记得文章一开始，我们说32位机器下，OC中的 `BOOL` 类型实际是 `signed char`，可以表示-128~127。在执行到 `check` 方法的第一句时，由于 `[self returnValue]` 返回的是int类型的数值，所以会进行以下的 `int` -> `signed char（BOOL）`类型转换： 
	
	BOOL yesOrNo = (BOOL)256;
而 `signed char` 占一个字节，也就是8 bit。 256转换成二进制的末16位是 `00000001 00000000`，强制转换为 `signed char` 后，取末8位的值 `00000000`，也就是0。到这里，大家应该意识到，所有末8位全为0的数字，如512、1024...，在32位机器上被强制转换位 `BOOL` 类型时，都会被转换位 `NO(0)`。之所以会发生这种“神奇的现象”，是因为32位机器上的 `BOOL` 类型在“作祟”。  
指针类型也一样。继续拿上面的代码举例，这次我们把 `returnValue` 方法改造下：

	- (NSString *)returnValue {
		NSString *someString = @"xxx";
    	return someString;
	}
	
如果 `someString` 为nil，那么没有疑问 `check`方法会输出 NO。如果 `someString` 不为nil呢？我们假设 `someString` 的内存地址为  `0xa3b2c100`,在将 `returnValue` 强制转换为 `yesOrNo` 时，仍然取内存地址的末8位，也就是 `0x00`，所以yesOrNo的值就是 `NO` 喽。 当然，只要原字符串的内存地址末8位不全是0，比如`0xa3b2c1d0` 等，转换后yesOrNo的值就不会为`NO`。

---  
  
上面说的都是32位机器上才会出现的问题，64位机器上，OC中的BOOL成了真正C语言中的bool，只能存储0和1。所以，在进行类似的类型转换时，只要值不等于0，就会被判为1，不会出现这些问题。但是由于目前iOS仍然需要支持32位的机器，所以平常写代码时还得多多注意。  

#### _32位的iOS设备：iPhone4s(A5)、iPhone5(A6)、iPhone5c(A6)、iPod Touch5(A5)、iPad3(A5X)、iPad4（A6X)_  

---

参考链接：  
- [https://www.bignerdranch.com/blog/bools-sharp-corners/](https://www.bignerdranch.com/blog/bools-sharp-corners/)  
- [https://en.wikipedia.org/wiki/List_of_iOS_devices](https://en.wikipedia.org/wiki/List_of_iOS_devices)



