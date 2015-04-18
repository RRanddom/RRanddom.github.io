---
layout : defalut
title : Swift中optional介绍
---

## Optional是什么

在C语言中，我们可以"声明"一个变量而不用给它初始值.比如

```C
 int number

```

如果你在number没有初始化时使用它,会有意想不到的结果

相反,在Swift世界里,为了安全,需要所有变量和常量有一个值.这样可以防止使用未初始化变量(常量)产生的不可预知的结果.不过,有的时候我们声明一个变量,它有可能是某个对象,又有可能是nil.举个例子:我们搜索的时候,搜索结果可能为空

## Optional是怎么定义的

为了解决这个问题,Swift 创建了Optional这个_类型_,它即可以表示某个对象,也可以是nil

```Swift
enum Optional {
    case None
    case Some<T>

```
你通过加一个'?'来声明一个optional类型 (String?)

## 展开一个可选值 (Unwrapping an Optional)

当你使用Optional前,你先得unwrap它,就是说你在用它前先要断言它必定不是空值,我们把optional对象看成是对象本身和null值的绑定

### Optional Binding

你可以用两种方式展开(unwrap)一个optional值,安全的方式是使用 Optional Binding

```Swift
let possibleString :String? = "Hello"
if let actualString = possibleString {
    println(actualString)
} else {
    // if possibleString is nil
}

```

### Forced Unwrapping (强行展开)

有事你很确信某Optional value是非空的,你可以用'!'来强行展开

```Swift
let possibleString : String? = "Hell0"
println(possibleString?)
```

如果possibleString是空值，这个程序就崩了

### Implicitly Unwrapped Optional (隐式展开的Optional)

隐式展开是指Optional类型不用展开(它悄悄自己展开了).这些品种的Optional类型用!定义(不是?)

```Swift
let possibleString : String! 
println(possibleString)
```
和强行展开一样,如果possibleString是空值,程序就崩了


