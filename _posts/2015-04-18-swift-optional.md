---
layout : default
title : Swift中0ptional介绍
---

## Optional是什么

在C语言中，我们可以"声明"一个变量而不用给它初始值.比如

```C
 int number

```

如果你在number没有初始化时使用它,会有意想不到的结果

相反,在Swift世界里,为了安全,需要所有变量和常量有一个值.这样可以防止使用未初始化变量(常量)产生的不可预知的结果.不过,有的时候我们声明一个变量,它有可能是某个对象,又有可能是nil.举个例子:我们搜索的时候,搜索结果可能为空

## Optional是怎么定义的

为了解决这个问题,Swift 创建了Optional这个_类型_,它既可以表示某个对象,也可以是nil

```Swift
enum Optional {
    case None
    case Some<T>
}
```
你通过加一个'?'来声明一个optional类型 (String?)

## 展开一个可选值 (Unwrapping an Optional)

当你使用Optional前,你先得unwrap它,就是说你在用它前先要判断它会不会不是空值,我们把optional对象看成是对象本身和null值的绑定(或者说是复合)

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

有时你很确信某Optional value是非空的,你可以用'!'来强行展开

```Swift
let possibleString : String? = "Hell0"
println(possibleString!)
```

如果possibleString是空值，这个程序就崩了

### Implicitly Unwrapped Optional (隐式展开的Optional)

隐式展开是指Optional类型不用展开(它悄悄自己展开了).这些品种的Optional类型用!定义(不是?)

```Swift
let possibleString : String! 
println(possibleString)
```
和强行展开一样,如果possibleString是空值,程序就崩了

## Implicitly Unwrapped Optionals in Swift

## 什么时候使用隐式可选类型(Implicitly Unwrapped Optional)

### 1. 在初始化时不能确定的常量

每一个常量在初始化结束时都得有一个值.有时,初始化过程中没法给常量成员赋值,不过依然能保证用到它的时候它能有一个值

使用可选类型(Optional)会有一个问题,初始化时会给常量成员赋值<code>nil</code>,接着,常量成员是不可变的···当你知道你个可选类型一定非空的时候,每次unwrap那个成员就显得很啰嗦.使用隐式可选类型就好啦,用的时候不必每次都unwrap

有一个很好的例子,在view加载完成前,UIView的某个子类没办法初始化

```Swift
class MyView : UIView {
    @IBOutlet var button : UIButton
    var buttonOriginalWidth : CGFloat!
    
    override func viewDidLoad() {
        self.buttonOriginalWidth = self.button.frame.size.width
    }
}

```

你在view加载完成前,你不可能知道original width的值,而且你知道<code>viewDidLoad</code>一定会先于其他方法执行.你可以将buttonOriginalWidth声明为隐式可选类型,这样可以省去很多unwrap的代码
 
### 2. 和0bjective-C API交互

我们使用Objective-C中的对象时,都是获取指向该对象的指针,指针很可能是空的(对象被释放了).意味着,你在Swift中操作Objective-C的API时必须使用可选类型(来指向一个对象).你可以使用普通的可选类型,如果你知道某个对象一定是非空的,那不妨把它声明为隐式可选类型

一个例子

```Swift
override func tableView(tableView: UITableView!, cellForRowAtIndexPath indexPath: NSIndexPath!) -> UITableViewCell? { return nil }

```
如果没有tableView对象或是indexPath对象.这个方法也不会被调用.所以检查UITableView或是indexPath是浪费时间

### 3. 当某个变量为nil时你的App不能继续运行了

很稀有的情况,不过假如某个对象为空时,你的App就不能运行,那就没必要检查那个对象是否为空了.如果你需要检查某个值来确保App能正确运行,使用断言(assert)就好了.隐式可选类型内建了一个检查是否为nil的断言

### 4. NSObject的初始化

技术上讲,所有继承自NSObject的类的构造器都会返回一个隐式可选类型的对象.这是因为Objective-C的构造方法会返回nil.意味着某些情况下,你需要手动检查构造函数的返回是不是nil.举个例子,当image不存在时,UIImage就会返回nil

```Swift

var image : UIImage? = UIImage(named: "NonExistentImage")
if image.hasValue {
    println("image exists")
}
else {
    println("image does not exist")
}

```

如果你认为有可能Image不存在,并且你能优雅地处理这种情况,你可以声明(捕获到的)构造函数的返回值为可选类型.当然你也可以选隐式可选类型(这里还是用普通的Optional比较好)

## 什么时候不用隐式可选类型

1. 惰性计算的成员变量

有时,有些成员变量永远不应该是nil,不过在初始化时没办法正确赋值.一种方法是声明它为隐式可选,另一种方法是声明为惰性变量(lazy variable)

```Swift
class FileSystemItem {
}

class Directory : FileSystemItem {
    @lazy var contents : [FileSystemItem] = {
        var loadedContents = [FileSystemItem]()
        // load contents and append to loadedContents
        return loadedContents
    }()
}
```
contents在它第一次没调用前一直都没初始化.

> 注意: 它可能与上面那个buttonOriginalWidth的例子混淆.
> 但是buttonOriginalWidth值必须在viewDidLoad方法中被初始化
> 在此之前,buttonOriginalWidth不会被调用

2. 其他情况都不该用

应该尽可能少用隐式可选类型,因为一旦用错(accessed while nil),整个App就Crash了,使用可选类型会保险一点


翻译自[这里](http://www.drewag.me/posts/what-is-an-optional-in-swift#optional-binding)
