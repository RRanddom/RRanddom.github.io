---
layout : default
title : Effective-Objective-C-笔记
---

### Item 3

#### Literal Arrays

创建Array有两种方式

```Objective-C
//方法A
NSArray *arraya = [NSArray arrayWithObjects: objc1,objc2,objc3,nil];
//方法B
NSArray *arrayb = @[objc1,objc2,objc3];
```

如果objc2为nil时，arrayb在创建过程中会抛出异常，而arraya不会，但是arraya中只会有一个元素。

### Item 4

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

### Item 5

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

// 2015-08-12 16:18:45

### Item 6 : 理解Property

@dynamic 的作用，告诉compiler不要自动创建实例，不要自动创建accessors。当编译器发现你access那些用dynamic修饰的变量时，它不会抛出异常，它相信在运行时这些值就available了

一个例子:

```Objective-C

@interface EOCPerson : NSManagedObject
@property NSString * firstName;
@property NSString * lastName;
@end

@implementation EOCPerson
@dynamic firstName,lastName;
@end

```

strong 和 weak的区别，strong是一种强的ownership。当该属性被赋值的时候，新值先被retain(引用计数+1)，旧的值被release，然后完成赋值

weak是一种弱的ownership，被赋值的时候，新值没有被retain，就的值也没有被release，和assign差不多。

copy和strong差不多。不过当属性被赋值的时候，新值被拷贝了。对NSString很管用，加入一个属性被一个MutableString赋值了，那个MutableString在别的地方被修改了，这个修改可能出乎你的意料，所以copy的作用就是在赋值String的时候创建一个不可变的String。

再来看一段代码:

```Objective-C
@interface EOCPerson :NSManagedObject

@property (copy) NSString * firstName;
@property (copy) NSString * lastName;

- (id) initWithFirstName:(NSString *)firstName
                lastName:(NSString *)lastName;

\\ implementation

- (id)initWithFirstName:(NSString *)firstName
               lastName:(NSString *)lastName
{
  if(self = [super init]) {
    _firstName = [firstName copy];
    _lastName = [lastName copy];
  }
  return self;
}

@end

```

为什么不用setter方法直接给firstName赋值(self.firstName = XXX)。你在init或者dealloc方法中永远不能用accessor(就是不能调用property的getter&setter)。

一个更好的方式:

```Objective-C
@property (copy,readonly) NSString * firstName;
@property (copy,readonly) NSString * lastName;

//没有定义setter，(我们不需要)
```

atomic修饰的属性能确保读写的原子性，不多说了。

小测验:为啥需要@property这个关键词，


### Item 7:

OC社区一直有个分歧:有些人觉得使用dot syntax来访问一个实例，有些人觉得直接访问实例变量会好一点，笔者觉得读的时候应当直接访问，写的时候借助dot syntax

考虑这个例子:

```Objective-C
@interface EOCPerson: NSObject
@property (nonatomic,copy) NSString * firstName;
@property (nonatomic,copy) NSString * lastName;

- (NSString *)fullName;
- (void)setFullName:(NSString *)fullName;
@end

/// fullName & setFullName的实现可能如下

- (NSString *)fullName {
  return [NSString stringWithFormat:@"%@%@,
          self.firstName,self.lastName"];
}

- (void)setFullName:(NSString *)fullName {
  NSArray * components = [fullName componentsSeparatedByString:@""];
  self.firstName = [components objectAtIndex:0];
  self.lastName = [components objectAtIndex:1];
}

```

在getter&setter里面，我们都通过dot syntax来读&写,再来看另一种写法

```Objective-C
- (NSString *)fullName {
  return [NSString stringWithFormat:@"%@%@",
          _firstName,_lastName];
}

- (void)setFullName:(NSString *)fullName {
  NSArray * components = [fullName componentsSeparatedByString:@""];
  _firstName = [components objectAtIndex:0];
  _lastName = [components objectAtIndex:1];
}

```

两者有轻微的不同，直接访问速度快，直接访问会跳过property setter定义的内存管理方式。举例：如果property是copy修饰的，直接赋值会导致不会有copy发生，新的值被retain了，旧的被release

KVO的消息也不会发送了 -- 这个其实挺重要你懂的

一些警告:在初始化方法里面赋值的时候，永远直接赋值，因为子类化可能直接重写了setter方法。假设有个类EOCSmithPerson继承了EOCPerson。它重写了setter

```Objective-C
- (void) setLastName:(NSString *)lastName {
  if(![lastName isEqualToString:@"Smith"]) {
    [NSException raise:NSInvalidArgumentException format:@"Last name must be Smith"];
  }
  self.lastName = lastName;
}
```

有些场景下你必须在初始化方法中使用setter：变量实在父类中定义的，你直接看不到

另一个警告:你的property使用懒加载。如果你不使用getter方法，那个变量就一直是空的，(应该都遇到过)

```Objective-C
- (EOCBrain *)brain {
  if(!_brain) {
    _brain = [Brain new];
  }
  return _brain;
}
```

### Item 8 : 理解对象之间的比较

直接上代码

```Objective-C
NSString * foo = @"Bardger 123";
NSString * bar = [NSString stringWithFormat@"Bardger %i",123];
BOOL equalA = (foo == bar);
BOOL equalB = [foo isEqual:bar];
BOOL equalC = [foo isEqualToString:bar];

```

这一节没什么难的，pass

// 2015-08-14 02:38:26

### Item 10 : Associated Objects

有下列方法:

```Objective-C

void objc_setAssociatedObject(id object,void *key,id value,objc_AssociationPolicy policy)

id objc_getAssociatedObject(id object,void key)

void objc_removeAssociatedObjects(id object)

```

第一个方法很像[object setObject:value forKey:key] (object 是NSDictionary类型)，第二个方法像[object objectForKey:key]

举个使用association的例子:

在iOS开发中，会经常使用UIAlertView(我推荐使用UIAlertController)。比如这样:

```Objective-C
- (void)askUserAQuestion {
  UIAlertView * alert = [[UIAlertView alloc] initWithTitle:@"Question"
                                              message:@"What do you want to do?"
                                              delegate:self
                                              cancelButtonTitle:@"Cancel"
                                              otherButtonTitles:@"Continue",nil];

  [alert show];
}  
  ///delegate method
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSUInteger)buttonIndex
{
  if(buttonIndex == 0) {
    [self doCancel];
  } else {
    [self doContinue];
  }
}

```

当你个页面有多个alertView的时候就有些麻烦了，delegate方法先要判断是那个alertView被触发了。如果在触发某个alertView的时候，alertView每个选项的逻辑能被确定(而不是在delegate方法中才被确定),那就好多了。

```Objective-C
#import <objc/runtime.h>

static void *EOCMyAlertViewKey = "EOCMyAlertViewKey";

- (void)askUserAQuestion {
  UIAlertView * alert = [[UIAlertView alloc] initWithTitle:@"Question"
                                              message:@"What do you want to do?"
                                              delegate:self
                                              cancelButtonTitle:@"Cancel"
                                              otherButtonTitles@"Continue",nil];


  void (^block)(NSInteger) = ^(NSInteger buttonIndex) { // block接受一个int，什么都不返回
    if (buttonIndex == 0) {
      [self doCancel];
    } else {
      [self doContinue];
    }
  };

  objc_setAssociatedObject(alert,
                      EOCMyAlertViewKey,
                      block,
                      OBJC_ASSOCIATION_COPY);

 [alert show];
 }

 ///delegate method
 - (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex
{
  void (^block)(NSInteger) = objc_getAssociatedObject(alertView,EOCMyAlertViewKey);
  block(buttonIndex);
}

```

### Item 11: objc_msgSend

我们知道，在C语言里面，函数调用是通过一种叫做static binding的技术，我们调用的函数在编译期就已经确定了，函数的地址是hardcode到指令流里面的。

```C

#import <stdio.h>

void printHello() {
  printf("Hello,world\n");
}

void printGoodbye() {
  printf("Goodbye,world");
}

void doXXX (int type) {
  void (*fuc)();
  if(type == 0) {
    fnc = printHello;
  }else {
    fnc = printGoodbye;
  }
  fuc();
  return 0;
}
```

这儿用的是dynamic binding，懒得讲了，自己翻书切。

dynamic binding 是oc里面函数调用使用的技术，OC里面方法调用--> id retVal = [someObject messageName:param];

编译器把它翻译成C function --> id retVal = objc_msgSend(someObject,@selector(messageName:),param);

### Item 12 : 理解Message Forwarding

这一节主要讲当object不能识别发送给它的消息时会发生什么。嗯，会发生一种叫做Message forwarding的东西:

看下面这段提示

```
[_NSCFNumber lowercaseString]:unrecognized selector sent to instance 0x87
*** Terminating app due to uncaught exception
'NSInvalidArgumentException',reason:'[_NSCFNumber lowercaseString]:unrecognized selector sent to instance 0x87'
```

它是NSObject类抛出的一个异常，叫做 doesNotRecognizeSelector:

剩下的有些看不懂.

### Item 13 : 用Method Swizzling来debug 看不要具体实现的方法

调用的某个方法可以在运行时更改，可以改变我们看不到实现的方法。

```Objective-C
// 交换Method
void method_exchangeImplementations(Method m1,Method m2)

// 获取Method
Method class_getInstanceMethod(Class aClass, SEL aSelector)
```

下列代码交换了lowercaseString 和uppercaseString的实现

```Objective-C

Method originalMethod = class_getInstanceMethod([NSString class],
                                        @selector(lowercaseString));

Method swappedMethod = class_getInstanceMethod([NSString class],
                                          @selector(uppercaseString));

method_exchangeImplementations(originalMethod,swappedMethod);
```

然后，下次调用lowercaseString，其实调用的是uppercaseString，调用uppercaseString，其实调用的是lowercaseString。

假如你想在调用lowercaseString方法的时候，顺便打印出结果，怎么搞?

```Objective-C

@implementation NSString (EOCMyAddtions)
- (NSString *)eoc_myLowercaseString {
  NSString * lowercase = [self eoc_myLowercaseString];
  NSLog(@"%@=>%@",self,lowercase);
  return lowercase;
}
@end
```

看上去是递归，但其实不是的，运行时，eoc_myLowercaseString会被替换成lowercaseString。

```Objective-C

Method originalMethod = class_getInstanceMethod([NSString class],
                                          @selector(lowercaseString));

Method swappedMethod = class_getInstanceMethod([NSString class],
                                          @selector(eoc_myLowercaseString));

method_exchangeImplementations(originalMethod,swappedMethod);

//然后

NSString * string = @"ThIs iS tHe sTriNg";
NSString * lowercase = [string lowercaseString]; // 能自动打印XXX

```
这是大杀器，慎用!!!

### Item 14 : 理解什么是Class Object

typedef struct objc_object {
  Class isa;
} * id;

```Objective-C
typedef struct objc_class * Class;
struct objc_class {
  Class isa; //
  Class super_class;
  const char *name;
  long version;
  long info;
  long instance_size;
  struct objc_ivar_list *ivars;
  struct objc_method_list **methodLists;
  struct objc_cache *cache;
  struct objc_protocol_list *protocols;
}

```
OC的introspection机制就是靠isa指针来获取对象的类，然后顺着继承关系(寻找父类)，如果没有introspection机制，你就没法完全了解一个类。

```Objective-C

//比较object是否是EOCSomeClass类型的:
id object = /**/
if([object class] == [EOCSomeClass class]) {
  //
}
//这种方法不好，优先使用introspection机制
```
//2015-08-17 14:06:18

### Item 16: desinated initializer

desinated initializer 是指一个特别的构造器，其他的构造方法都会调用它。举个例子:

```Objective-C
// NSDate
-（id)init;
- (id)initWithString:(NSString *)string
- (id)initWithTimeIntervalSinceNow:(NSTimeInterval)seconds
                                  sinceDate:(NSDate *)refDate
- (id)initWithTimeIntervalSinceReferenceDate:
                          (NSTimeInterval)seconds
- (id)initWithTimeIntervalSince1970:(NSTimeInterval)seconds
```

其中initWithTimeIntervalSinceReferenceDate:就是desinated initializer。

没什么好讲的，哼

### Item 18 : 选择不可变的对象

当你设计一个类的时候，你肯定会使用很多property来保存数据。建议只有当必须的时候才将对象变为可变的。(尽量将property标记为readonly)

例子

```Objective-C
// 从webservice去到的POI数据，通常不需要任何修改
@interface EOCPointOfInterest: NSObject

@property (nonatomic,copy,readonly) NSString * identifier;
@property (nonatomic,copy,readonly) NSString * title;
@property (nonatomic,assign,readonly) float latitude;
@property (nonatomic,assign,readonly) float longtitude;

- (id) initWithIdentifier:(NSString *)identifier
                    title:(NSString *)title
                 latitude:(float)latitude
               longtitude:(float)longtitude

// 在category里面，把property定义为readwrite，这样类内可以修改property，类外不可以修改了

@interface EOCPointOfInterest()
@property (nonatomic,copy,readwrite) NSString *identifier;
@property (nonatomic,copy,readwrite) NSString *title;
@property (nonatomic,assign,readwrite) float latitude;
@property (nonatomic,assign,readwrite) float longtitude;

@end
```

再举个例子，假如有个类，它负责存person的名字和他的朋友，

```Objective-C
@interface EOCPerson : NSObject

@property (nonatomic,copy,readonly) NSString * firstName;
@property (nonatomic,copy,readonly) NSString * lastName;
@property (nonatomic,strong,readonly) NSSet * friends

- (id) initWithFirstName:(NSString *)firstName
                andLastName:(NSString *)lastName;
- (void)addFriend:(EOCPerson *)person;
- (void)removeFriend:(EOCPerson *)person;
@end

// .m file

@interface EOCPerson()
@property (nonatomic,copy,readwrite) NSString *firstName;
@property (nonatomic,copy,readwrite) NSString *lastName;
@end

@implementation EOCPerson {
  NSMutableSet *_internalFriends;
}

- (NSSet *)friends {
  return [_internalFriends copy];
}

- (void)addFriend:(EOCPerson *)person {
  [_internalFriends addObject:person];
}

- (void)removeFriend:(EOCPerson *)person {
  [_internalFriends removeObject:person];
}

- (id)initWithXXXX { // 懒得手打了
  if((self = [super init])) {
    _firstName = firstName;
    _lastName = lastName;
    _internalFriends = [NSMutableSet new];
  }
  return self;
}
@end
```

你可以将friends用mutableset来修饰，然后自己操作friends这个set来添加/删除元素，不过数据间耦合度太低了，容易产生bug。

### Item 20 : 命名规则

记住两点 :

* private方法用小写+下划线作为前缀，比如 p_private_method
* 不要单单使用一个下划线作前缀，可能会重写系统的方法

### Item 21 : 理解Objective-C错误处理方式

NSError的用法

* 用法一:通过delegate方法来传递  //异步

* 用法二:通过 - (void)doSomething:(NSError ** )error //同步

```Objective-C
- (BOOL)doSomething:(NSError *)error {
  // do sth that cause an error
  if (/*there is an error*/) {
    if(error) {
      *error = [NSError errorWithDomainXXX];
    }
    return NO;
  }else {
    retain YES;
  }
}

总结:对于致命错误，抛出NSException,非致命错误通过NSError来处理

```

### Item 22 : 理解NSCopying协议

关于copy & mutableCopy，看[这篇文章](https://www.zybuluo.com/MicroCai/note/50592),基本涵盖了这个Item的内容

### Item 23 : Protocols

protocols主要用来实现delegate模式，但也有其他用法

### Item 24 : 使用category

差不多都会

### Item 25 : category 需要前缀

比如

```Objective-C
// extension本身不需要加前缀，如果有两个extension同名，会有warning不会有error
@interface NSString(HTTP)

// 加abc前缀 防止override
- (NSString *)abc_urlEncodedString;

- (NSString *)abc_urlDecodeString;
```

### Item 26 : 别在category里面定义property

错误示范

```Objective-C
@interface EOCPerson : NSObject

@property (nonatomic,copy,readonly) NSString *firstName;
@property (nonatomic,copy,readonly) NSString *lastName;

- (id)initWithFirstName:(NSString *)firstName;
               lastName:(NSString *)lastName;
@end

@implementation EOCPerson
@end

@interface EOCPerson(Friendship)
@property (nonatomic,strong) NSArray *firends;
- (BOOL)isFriendsWith:(EOCPerson *)person;
@end


@implementation EOCPerson(Friendship)
@end
```

编译会得到类似的warning

```
warning:property 'friends' requires method 'friends' to be defined - use @dynamic or provide a method implementation in this category [...]
```

意思是说friends没法合成getter&setter。

如果要强行加property，你得这么做!

```Objective-C

#import <objc/runtime.h>

static const char *kFriendsPropertyKey = 'kFriendsPropertyKey';

@implementation EOCPerson(Friendship)

- (NSArray *)friends {
  return objc_getAssociatedObject(self,kFriendsPropertyKey);
}

- (void)setFriends:(NSArray *)friends {
  objc_setAssociatedObject(self,kFriendsPropertyKey,friends,OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
@end
```

易出错、不推荐。

不过可以这样做

```Objective-C

@interface NSCalendar (EOC_Addtions)
@property (nonatomic,strong,readonly) NSArray *eoc_allMonths;
@end

@implementation NSCalendar (EOC_Addtions)

- (NSArray *)eoc_allMonths {
  return @[@"January",XXX];
}
```

因为该属性是只读的，不必为此生成setter方法，而且从getter方法看，不要要一个instance来存任何数据，其实这样也不好，直接写一个方法能达到同样的效果

```Objective-C

@interface NSCalendar(EOC_Addtions)
- (NSArray *)eoc_allMonths;
@end

```

> 关于内存管理的内容，类似文章实在是太多了，我准备另外写一篇文章记录一下

### Item 37 : 理解Block

下列内容主要是关于Block和多线程的，

关于block的常用语法，可以看[这个网站](http://fuckingblocksyntax.com/)，非常管用，swift版的闭包可以看[这个网站](http://fuckingswiftblocksyntax.com/)，主要用来备忘。

不多说了，重要的一点是，block会capture the variables in the scope which it is declared，不过它不能改变那些变量，否则会报错，可以用<code>\_\_block</code>定义那些可以被block改变的变量。

```Objective-C

NSArray * array = @[@0,@1,@2,@3,@4];
__block NSInteger count = 0;
[array enumerateObjectUsingBlock:
        ^(NSNumber *number,NSUInteger idx,BOOL *stop) {
          if ([number compare:@2] == NSOrderedAscending) {
            count ++;
          }
          }];
```

//count == 2

对了，有时候self会被implicitly captured by block，比如

```Objective-C
@interface EOCSomeClass

- (void) anInstanceMethod {
  void (^someBlock)() = ^{
    _ansInstanceVariable = @"Sth";
    NSLog(@"%@",_ansInstanceVariable)
  };
}
```

对_ansInstanceVariable赋值其实相当于 <code>self->\_ansInstanceVariable = @"sth"</code>

可能引起retain cycles。

>算了，我仔细想了想，block和多线程也自成一章吧
