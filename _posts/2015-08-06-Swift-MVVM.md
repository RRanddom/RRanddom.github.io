---
layout : default
title : MVC -> Swift MVVM
---

这是一篇翻译的文章，最近在学Swift!

这篇文章不是讲"MVVM大法好"或者"别用MVC，快切换到MVVM吧"，它主要讲在Swift语境下如何完善MVC模式

为了区分，先解释下文中的view是什么意思。view是呈现数据的一整块或一小块屏幕。比如：由一个image view 和一个title label组成的collection view cell，或是由title label、日期label和文本label组成的一个view。反正我说的view不是狭义的UIView

假如需要一个view来展示一篇文章。view应当呈现文章的主体文本、标题和发表时间等元素。在UIKit中，view和controller之间是强耦合的(所以才命名为view controller)，因
为我们完全不在乎view的布局，所以我们直接把view定义为view controller。

```Swift

class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel
}

```

然后，我们需要一些数据来填充view。我们先定义一个Article Model

```Swift

class Article {
  var title: String
  var body: String
  var date: NSDate
  var thumbnail: NSURL
  var saved: Bool
}

```

剩下的就是填充view了，我们需要的textview、标题和日期在model中都有了。在MVC架构中，那是controller的工作。

```Swift

class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel

  var article: Article {
    didSet {
      titleLabel.text = article.title
      bodyTextView.text = article.body

      let dateFormatter = NSDateFormatter()
      dateFormatter.dateStyle = NSDateFormatterStyle.ShortStyle
      dateLabel.text = dateFormatter.stringFromDate(article.date)
    }
  }
}

```
我们使用Swift的[property observer](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Properties.html#//apple_ref/doc/uid/TP40014097-CH14-XID_390)，使得每次property值改变的时候，didSet里面的代码都能执行(不过在构造方法中，那个方法不会被执行)。每当property改变，view就会有新的数据填充。我们还需要对model进行一些处理，Date标签需要String来初始化，不过model提供的是NSdate对象

现在考虑这么个场景，你完全实现了ArticleViewController，现在你还需要支持其他类型的article model，比如说字典类型的article

```Swift
{
  "title": "<string>",
  "body": "<string>",
  "date": "<json_date>",
  "thumbnail": "<url>"
}
```

你会怎么做呢?一个简单的方法是定义另外一个property，就叫articleDictionary

```
class ArticleViewController: UIViewController {
  var article: Article? {
    didSet {
      if let article = article {

      }
    }
  }

  var articleDictionary: [String:AnyObject]? {
    didSet {
      if let article = articleAsDictionary {
        titleLabel.text = artile["title"] as String?
        bodyTextView.text = artile["do"] as String?

        if let jsonDate = artile["date"] as String? {
          dateLabel.text = /*parse and format date from jsonDate */
        }
      }
    }
  }
  // ...
  // any subsequent interaction with the model would result in:
  //
  // if let article = article {
  //  proceed as article as Article
  // } else if let article = articleAsDictionary {
  //  proceed as article as Dictionary
  // } else {
  //   handle no article case
  // }
  // ...
}

```

永远别这么做！你会需要很多的if-else来处理model，会导致代码变得很复杂，难以阅读，容易出bug

一个比较好的idea是将dictionary转化为Article对象。但如果Article是Core data中的managed object就没辙了?如果没有managed object context就没法实例化它。另外，定义个所有Article遵守的protocol
也不是一个好主意。我们需要新的方法。

在讨论解决方法之前，我们先考虑一个新的问题，我们的controller非常简单，但实际项目中会非常复杂。记得我们要先操作NSDate，才能获得用户看得懂的string格式?对于你可能不是什么难事，不过如果我们还需要展示之前
定义的photo。我们的model没有image，而是image的url，我们的controller还需要网络请求和图片处理逻辑。在想想controller还要作什么：它会监听屏幕旋转，调整view，处理button点击事件和滑动事件，还有个手势，保存状态。。。如果能避免，就不应当让controller处理model。这是我们用MVVM模式处理的第二个问题。

在架构设计的过程中我们的目标永远是组件的简单化。一般通过限制组件的输入和职责来达到这个目标。我们就要对controller做这样的事情。我们把处理model的职责分离开来，指定controller的输入状态。
最优雅的做法是定义一个指定view显示的protocol。那就是我们view model

```Swift
protocol ArticleViewViewModel {
  var title: String { get }
  var body: String { get }
  var date: String { get }
}

```
如你所见，view model protocol定义了view所需要的所有输入，都是string类型的，property只提供get方法，我们在controller里面不能对他们进行修改。

MVVM全称是Model-View-ViewModel。我们现在构建的是Model-View-ViewController-ViewModel，不过view和controller紧密耦合在一起了，从MVVM的角度看，view controller是view的一个部分。我们
为了表达简单，还是把它称为MVVM吧。

让我们重构一下article view controller

```Swift
class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel

  var viewModel: ArticleViewViewModel {
    didSet {
      titleLabel.text = viewModel.title
      bodyTextView.text = viewModel.body
      dateLabel.text = viewModel.date
    }
  }
}
```

美吗?我们做的就是将viewModel的property匹配到UI元素上，我们将view controller和model独立开来了，把丑陋的data formmatter和model类型检查都抛弃掉了。很棒是不是，不过我们仍然需要搞到真实的数据源。
你可能猜到了，在view model的具体实现里面有。

具体实现view model有几种方法，如果你熟悉Swift的[extensions]()，你可以拓展Article来遵守ArticleViewViewModel协议。那是很棒的解决方案，不过当Article是字典时没用。我们创建一个新类，它实现了ArticleViewViewModel
，需要Article对象作为输入

```Swift

class ArticleViewViewModelFromArticle: ArticleViewViewModel {
  let article: Article
  let title: String
  let body: String
  let date: String

  init(_ article: Article) {
    self.article = article

    self.title = article.title
    self.body = article.body

    let dateFormatter = NSDateFormatter()
    dateFormatter.dateStyle = NSDateFormatterStyle.ShortStyle
    self.date = dateFormatter.stringFromDate(article.date)
  }
}

```
然后我们

```Swift

let article: Article = /* some article*/
let viewModel = ArticleViewViewModelFromArticle(article)
let viewController = ArticleViewController()
viewController.viewModel = viewModel
```

现在controller和model完全解耦了，controller接受一个viewModel，viewModel遵守某个协议，所有的model操作全在view model里面实现，controller支持任何种类的view model对象，只要该对象遵守了view mode协议

我们实现的方案适用于某些场合，为啥?你可能注意到了，我们view model里面定义的property是用let定义的，是常量而非变量。我们试着加一个thumbnail到我们的Article view里面去。Thumbnail需要一个imageView
来显示。Image View接受一个UIImage作为输入，我们的model里面只有一个URL，我们该怎么办呢？我们在view model的协议中提供thumbnail Image

```Swift

protocol ArticleViewViewModel {
  var title: String { get }
  var body: String { get }
  var date: String { get }
  var thumbnail: UIImage? { get }
}
class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel
  var thumbnailImageView: UIImageView

  var viewModel: ArticleViewViewModel {
    didSet {
      titleLabel.text = viewModel.title
      bodyTextView.text = viewModel.body
      dateLabel.text = viewModel.date
      thumbnailImageView.image = viewModel.thumbnail
    }
  }
}

```

很简单，我们把image定义为Optional类型的，理由很简单(想想就知道了)

谁来提供thumbnail? 对，是view model。我们在view model里面加上下载thumbnail的方法。

```Swift
class ArticleViewViewModelFromArticle {
  let article: Article
  let title: String
  let body: String
  let date: String
  var thumbnail: UIImage?

  init(_ article: Article) {
    self.article = article

    self.title = article.title
    self.body = article.body

    let dateFormatter = NSDateFormatter()
    dateFormatter.dateStyle = NSDateFormatterStyle.ShortStyle
    self.date = dateFormatter.stringFromDate(article.date)

    let downloadTask = NSURLSession.sharedSession().downloadTaskWithURL(article.thumbnail) {
      [weak self] location, response, error in
      if let data = NSData(contentsOfURL: location) {
        if let image = UIImage(data: data) {
          self?.thumbnail = image
        }
      }
    }

    downloadTask.resume()
  }
}
```

它现在能工作吗? 不行!下载图片是一个异步的方法，thumbnail下载完的时候我们的view controller已经完成数据填充了，之后view model的任何改变都没什么卵用。

我们可以发送通知，然后让view controller监听这个通知，不过可拓展性不强，代码的可读性会变差，而且通知的注册和注销都需要额外的注意。另外一种方式就是通过delegate，然后让controller实现delegate。
比消息安全一点，不过引入了不必要的依赖，比消息的拓展性更差一点。这两个方法都不用，我们需要的是property级别的数据绑定技术。我们需要监听一个property，当它改变的时候收到通知。

我们可以使用KVO技术，当然对于Swift不太好用，代码很繁琐，不太好debug。

另一种技术就是ReactiveCocoa了。对于Swift来说也有点难用。

下一章再细说!
