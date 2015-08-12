---
layout : default
title : Effective-Objective-C-笔记
---

### Item3

#### Literal Arrays

创建Array有两种方式

```Objective-C
//方法A
NSArray *arraya = [NSArray arrayWithObjects: objc1,objc2,objc3,nil];
//方法B
NSArray *arrayb = @[objc1,objc2,objc3];
```

如果objc2为nil时，arrayb在创建过程中会抛出异常，而arraya不会，但是arraya中只会有一个元素。

### Item4

#### 常量

如果你要定义一个只在某个文件中用的常量，比如说kAnimationDuration，不要用#define,不要放在.h文件里面，在.m文件里面这么声明

```Objective-C

static const NSTimeInterval kAnimationDuration = 0.3

```
const 确保你试图修改变量时编译器能抛出错误，static能确保变量编译后是该编译单元的一个局部变量，如果没有声明static并且两个.m文件都有kAnimationDuration，就会出现duplicate symbol错误

加入说你想定义一个其他类也能使用的常量，比如NSNotification中的notification名字，你定义的常量就必须出现在全局的符号表里面。需要这么做

```Objective-C
// .h file
extern NSString * const EOCStringConstant;
// .m file
NSString *const EOCStringConstant = @"SomeValue"
```

我们定义了一个EOCStringConstant指针，它指向的地址不可改变。

头文件里面的extern关键词告诉compiler要为EOCStringConstant生成一个全局的symbol。

用这种方式定义常量的好处之一是，编译器不会允许对该常量的修改或是重定义，如果使用#define的话，可能会被重新定义

###Item5

#### Enum

enum有一种比较巧妙的用法，可以产生组合

```Objective-C
enum UIViewAutoresizing {
  UIViewAutoresizingNone = 0,
  UIViewAutoresizingFlexibleLeftMargin = 1<<0,
  UIViewAutoresizingFlexibleWidth = 1<<1,
  UIViewAutoresizingFlexibleRightMargin = 1<<2,
  UIViewAutoresizingFlexibleTopMargin = 1<<3,
  UIViewAutoresizingFlexibleHeight = 1<<4,
  UIViewAutoresizingFlexibleBottomMargin = 1<<5,
}
```

可以使用 OR来组合他们，比如UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight就代表宽高都可变。

对了，如果你要定义可以相互组合的Enum，这么搞:

```Objective-C

typedef NS_OPTIONS(NSUInteger,EOCPermittedDirection) {
  EOCPermittedDirectionUp = 1<<0,
  EOCPermittedDirectionDown = 1<<1,
  EOCPermittedDirectionLeft = 1<<2,
  EOCPermittedDirectionRight = 1<<3,
}
```

如果你不需要可组合的Enum，这么搞:

```Objective-C

typedef NS_ENUM(NSUInteger,EOCConnectionState) {
  EOCConnectionStateDisconnect,
  EOCConnectionStateConnecting,
  EOCConnectionStateConnected,
}
```


!!!值得注意的是，使用Switch语句处理一个enum的时候，不要写default，因为当你尝试在enum中加类型的时候，编译器不会给你警告，而是默默用default给你处理。
