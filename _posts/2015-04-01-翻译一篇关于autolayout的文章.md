---
layout : default
title : 翻译一篇关于tableView自动布局的文章
---

以前写的 (zjcblog.sinaapp.com)

# [翻译]定制的TableViewCell

原文是raywenderlich这篇[文章](http://www.raywenderlich.com/73602/dynamic-table-view-cell-height-auto-layout)

### 动态的Table View Cell高度和自动布局

在过去，你想创建一个有着高度可变的Cell的Table View，你就得写一大堆尺寸代码。你得算出cell内每个label、Image、Text Field的高度。

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWImageCell-Final-Pins.jpg)

诚然，这是非常艰难的，而且易犯错误。

在这个教程里面，你将学习到如何创建定制的Cell并且能自动计算他们的高度。如果你之前用过定制的cell，你很可能会向“哎呀，又要写很过涉及尺寸的代码啦。”

然而，如果我们跟你讲，不用写任何尺寸代码呢？

“吹牛逼！”你可能会说，不过其实是可以的。

在教程结束后，你将知道如何利用自动布局来减少你代码中的数百行sizing code。

~注意：这个教程适用于在Xcode5下开发iOS7的App，它不包含iOS 8 的新特性。

这个教程假设你懂一些基本的自动布局和UITableView的使用（包括它的数据来源的代理方法）。如果你需要重温这些知识点，可以看下面的几个链接。

- [视屏教程：开始自动布局](http://www.raywenderlich.com/?p=64392)
- [自动布局初级指南](http://www.raywenderlich.com/50317/beginning-auto-layout-tutorial-in-ios-7-part-1)
- [Table View系列视屏教程](http://www.raywenderlich.com/video-tutorials)

### 开始

iOS 6 引入了一个神奇的新技术：自动布局。开发者喜闻乐见，在街上载歌载舞；乐队甚至写歌赞扬这个新特性...

好啦，上面说的有些夸张，但这确实是个了不得的东西。

最初呢，自动布局是很难用的。手写自动布局代码仍然是Object-C代码冗长的根源之一。而且因为不能处理constraints，使用Interface Builder往往适得其反。

随着Xcode 5.1和iOS 7的发布，IB支持自动布局啦，这使得自动布局的易用性向前迈了一大步。不仅如此，iOS 7提供了一个非常重要的代理方法。

<pre class="brush: objc;">

- (CGFloat)tableView:(UITableView *)tableView estimatedHeightForRowAtIndexPath:(NSIndexPath *)indexPath;

</pre>

这个方法允许一个TableView惰性地计算每个Cell的实际高度，甚至是拖延到Table真正需要的时候才计算。

不过，你肯定不想趟入理论的泥潭，你可能想尽快写点代码，我们尽快来干点实事吧。

### APP概述

你想一下，你最重要的客户过来跟你说，“我们用户吵着要浏览他们最喜欢的Deviant Artists' submissions”（我也不知道是什么玩意儿）

你提过你根本就没听说过Deviant Art吗？

“嗯，这是一个流行的社交平台，艺术家们分享他们的作品，作品就被称为deviantion”。你的客户解释道。“你赶紧看一下一个网站。”

幸运的是，Deviant Art提供了一个RSS连接，你可以用它来获得某个艺术家的deviantion和post

“我们开始做应用了，不过我们卡在如何在Table view上显示内容”，你的客户承认，“你会么？”

突然间你。。。。。现在，你只需要一个称手的工具就能成为你客户心目中的英雄。

首先下载客户代码（这篇教程的起点），[here](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/DeviantArtBrowser_Starter.zip)。

这个项目使用了CocoaPods，打开*DeviantArtBrowser.xcworkspace* (不是.xcodeproj 文件).pods已经被包含了，所以你不需要运行 pod install命令。

（如果你需要学习有关CocoaPods的知识，你应当回顾下[这篇文章]()）

初始项目成功地下载了Deviant Art Rss的内容，但是没有显示他们，这就是你需要努力的地方了 :]

打开Main.storyboard 你会看到4个界面。

&emsp;&emsp; ![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/Main.jpg)

从左至右，他们是：

* 导航控制器。
* RWFeedViewController，标题是 Deviant Browser
* 两个RWDetailViewController的界面，一个显示纯文字内容，另一个显示文字加图像；他们的标题分别为Deviant Article和Deviant Media

构建、运行。你会看到console打印出一些东西，不过其他什么都没发生。

log看起来是这样的。

2014-05-28 00:52:01.588 DeviantArtBrowser[1191:60b] GET 'http://backend.deviantart.com/rss.xml?q=boost%3Apopular'

2014-05-28 00:52:03.144 DeviantArtBrowser[1191:60b] 200 'http://backend.deviantart.com/rss.xml?q=boost%3Apopular' [1.5568 s]

这个APP发起了一个Http Request，得到了一个响应，但是对返回的数据没有任何的处理。

现在，打开RWFeedViewController.m。这是大多数新代码创建的地方。让我们看看parseForQuery中的代码片段。

<pre class="brush: objc;">
[self.parser parseRSSFeed:RWDeviantArtBaseURLString
parameters:[self parametersForQuery:query]
success:^(RSSChannel *channel) {
[weakSelf convertItemsPropertiesToPlainText:channel.items];
[weakSelf setFeedItems:channel.items];

[weakSelf reloadTableViewContent];
[weakSelf hideProgressHUD];
} failure:^(NSError *error) {
[weakSelf hideProgressHUD];
NSLog(@"Error: %@", error);
}];

</pre>

self.parser是RSSParser的一个实例，它是MediaRSSParser的一部分。

这个方法向Deviant Art服务器发起一个请求来获得Rss feed，然后它返回一个RSSChannel到success block里面。格式化以后，(就是简单地把HTML转化为纯文本)，success把channel.items存储为一个叫做feedItems的property。

channel.items数组里面是RSSItem对象，每一个都代表了RSS feed的一个项。好啦，你现在知道需要Table View中需要显示什么啦；就是feedItems数组！

最后，你发现项目中有三个warning。它们是什么呢？

它看起来像有人手动放置了#warning 声明来提醒方法需要补充完整。嗯，这个方法挺方便的。

![](http://cdn1.raywenderlich.com/wp-content/uploads/2014/05/convenient.jpg)

### 创建基本的定制Cell

简单看过客户代码后，你发现App已经帮你完成了数据的取回，但是它还没有显示任何东西。为了显示，你首先需要创建定制的table view cell。

在DeviantArtBrowser 项目中添加新的类：RWBasicCell。让它继承UITableViewCell；确定它不会创建xib文件。

打开RWBasicCell.h文件，添加下面两个属性

<pre class="brush: objc;">
@property (nonatomic,weak) IBOutlet UILabel *titleLabel;
@property (nonatomic,weak) IBOutlet UILabel *subtitleLabel;
</pre>

接着打开Main.storyboard文件，拖动一个UITableViewCell到RWFeedViewController界面上去。

将那个cell的custom class设为RWBasicCell。

![](http://cdn1.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Class.jpg)

设Identifier为RWBasicCell。

![](http://cdn3.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Identifier.jpg)

设Row Height为83.

![](http://cdn4.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Row-Height.jpg)

拖一个UILabel到Cell中去，设它的文本为"Title"。

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Add-Title-Label.jpg)

设Title的Lines项(label可以有的最多的行数)为0，意味着无限制。

![](http://cdn3.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Title-Number-Lines.jpg)

设Title的的size项的位置和大小。

![](http://cdn4.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Title-Rect.jpg)

把代码中的Label和视图中的Label连接起来。

![](http://cdn3.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Title-Outlet.jpg)

接下来，再添加一个UILabel到Cell界面中去，就在title Label的正下方，将它的文本设为”Subtitle“。

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Add-Subtitle-Label.jpg)

和上面步骤类似的，改变subtitle的各种设置。。。。

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Subtitle-Rect.jpg)

将subtitle的Label颜色设为浅灰色(Light Gray Color)，字体为System 15.0，Line为0(个人更加喜欢Helvetica-Light这个字体)

![](http://cdn4.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Subtitle-Properties.jpg)

连接接代码中的subtitleLabel和界面中的subtitle。

![](http://cdn4.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Subtitle-Outlet.jpg)

哈哈，你已经设置完了RWBasicCell了。现在你只需要添加自动布局constraints了。选择叫做title的那个标签，把top，trailing，leading都设为20

![](http://cdn5.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Title-Pin-Constraints.jpg)

这确保了无论cell有多大，title标签永远都是：

* 距离顶端20个points
* 横跨整个cell，减去两边的20个points的空隙

现在选择subtitle便签，这次设置leading、buttom、trailing项为20 points

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Subtitle-Pin-Constraints.jpg)

原理跟上面的设置类似，它确保了subtitle总是处于cell的底端，横跨整个cell，旁边留出20个points的空隙。

专业意见：在UITableViewCell中自动布局的技巧就是保证有constraints来限制每个subview元素距离cell的边界有一定的距离，就是说，每一个subview应当由leading, top, trailing和bottom的限制。

更进一步，。。

棘手的问题在于，有时候你不填限制条件的时候，IB不会有warning产生....

面对这种情况，自动布局会在你运行程序的时候返回一个错误的高度。比如说，它会返回0作为cell高度，这就是说你的constraints设置错啦。

如果你在自己的项目中发现了错误，你就一直调节constraints就行了。

选择subtitle标签，按住control然后拖动到title标签上。选择 Vertical Spacing来连接subtitle的顶部和title标签的底部

然后你可能发现有一些warning，你要着急，慢慢fix。

对于之前设置的两个标签，设置Content Hugging Priority和Content Compression Resistance Priority中的水平方面和垂直方向的值为1000.

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Labels-Hugging-Compression-Resistance.jpg)

通过设置这些优先级为1000，这些constraints就会优于其他constraints。这告诉自动布局器，我们希望label首先能显示全它的文字。

最后，选择title标签，将它的Intrinsic Size从Default改成Placeholder。对于subtitle也这么干。

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWImageCell-Set-Intrinsic-Content-Size.jpg)

这告诉Interface Builder假设目前Label的大小就是他们实际的大小。这么做的话，constraints就无二义性了。

最后的最后，自动布局的constraints应该看起来是这样的。

![](http://cdn5.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Final-Constraints.jpg)

回顾：这些设置符合前面的自动布局标准吗？

1. 是否所有的subview都有关联所有边的constraints？ 是的
2. 这些constraints是否从cell的顶部开始，到cell的底部结束？ 是的

titleLabel距离顶部20个points，它距离subtitleLabel 0个points，subtitleLabel距离底部20个points

所以自动布局可以计算出cell的高度

接下来，我们需要创建从RWBasicCell到Deviant Article:的segue了。

选择RWBasicCell，然后按住control拖动到Deviant Article这个界面上，选择Push类型的segue

IB也将cell的Accessory属性自动改为_Disclosure Indicator_，看起来不太好，我们把它改回None。

![](http://cdn5.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Segue.jpg)

现在，用户无论何时点击一个RWBasicCell，app就会导航到RWDetailViewController。

鹅妹子嘤，RWBasicCell终于设置好了！如果你构建运行你的app，你就能看见...什么都没改变。怎么回事？

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/Say_What.jpg)

记得我们之前提到的#warning声明吗？那些东西就是问题所在。你需要实现table view的数据来源和代理方法。

### 实现UITableView的代理和数据来源

将下列语句添加到RWFeedViewController.m中，

<pre class="brush: objc;">
#import "RWBasicCell.h"
static NSString * const RWBasicCellIdentifier = @"RWBasicCell";
</pre>

在数据来源和代理方法中，你都会用到RWBasicCell这个类，而且你会通过RWBasicCellIdentifier来识别它。

接下来，我们补全数据来源方法。

将tableView:numberOfRowsInSection下的代码换成如下：

<pre class="brush: objc">
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
return [self.feedItems count];
}
</pre>

接着将tableView:cellForRowAtIndexPath：下的代码换成下列代码。

<pre class="brush: objc">

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
return [self basicCellAtIndexPath:indexPath];
}

- (RWBasicCell *)basicCellAtIndexPath:(NSIndexPath *)indexPath {
RWBasicCell *cell = [self.tableView dequeueReusableCellWithIdentifier:RWBasicCellIdentifier forIndexPath:indexPath];
[self configureBasicCell:cell atIndexPath:indexPath];
return cell;
}

- (void)configureBasicCell:(RWBasicCell *)cell atIndexPath:(NSIndexPath *)indexPath {
RSSItem *item = self.feedItems[indexPath.row];
[self setTitleForCell:cell item:item];
[self setSubtitleForCell:cell item:item];
}

- (void)setTitleForCell:(RWBasicCell *)cell item:(RSSItem *)item {
NSString *title = item.title ?: NSLocalizedString(@"[No Title]", nil);
[cell.titleLabel setText:title];
}

- (void)setSubtitleForCell:(RWBasicCell *)cell item:(RSSItem *)item {
NSString *subtitle = item.mediaText ?: item.mediaDescription;

// Some subtitles can be really long, so only display the
// first 200 characters
if (subtitle.length > 200) {
subtitle = [NSString stringWithFormat:@"%@...", [subtitle substringToIndex:200]];
}

[cell.subtitleLabel setText:subtitle];
}	
</pre>

上面代码的解释

*  在tableView:cellForRowAtIndexPath:这个方法里面，你可以调用basicCellAtIndexPath:获取一个RWBasicCell。接着你会看到，如果用一个help方法而不是直接返回一个cell的话，cell的拓展性会更强(这儿不是太懂)
* 在basicCellAtIndexPath:这个方法中，你取出一个RWBasicCell对象，用configureBasicCell:atIndexPath:方法设置它，最终，返回设置好的cell。
* configureBasicCell:atIndexPath:方法中，你获取indexPath位置的一个项(就是一个cell)，然后你就对titleLabel和subtitleLabel上的文字进行设置。

现在将tableView:heightForRowAtIndexPath:这个方法替换成下列代码：

<pre class="brush :objc">
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
return [self heightForBasicCellAtIndexPath:indexPath];
}

- (CGFloat)heightForBasicCellAtIndexPath:(NSIndexPath *)indexPath {
static RWBasicCell *sizingCell = nil;
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
sizingCell = [self.tableView dequeueReusableCellWithIdentifier:RWBasicCellIdentifier];
});

[self configureBasicCell:sizingCell atIndexPath:indexPath];
return [self calculateHeightForConfiguredSizingCell:sizingCell];
}

- (CGFloat)calculateHeightForConfiguredSizingCell:(UITableViewCell *)sizingCell {
[sizingCell setNeedsLayout];
[sizingCell layoutIfNeeded];

CGSize size = [sizingCell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
return size.height;
}
</pre>

最终你可以使用自动布局来计算cell的高度，

1. 在tableView:heightForRowAtIndexPath:这个方法中，它调用另外一个方法计算实际的高度heightForBasicCellAtIndexPath:，当然，这么做是为了可拓展性。

2. 你可能注意到了heightForBasicCellAtIndexPath:这个方法很有趣。

  * 这个方法使用GCD创建了一个sizingCell来保证它只能被创建一次。
  * 调用configureBasicCell:atIndexPath:来配置cell
  * 返回另一个方法calculateHeightForConfiguredSizingCell:，你可能已经猜到它也是为了可拓展性。

3. 最后，calculateHeightForConfiguredSizingCell:这个方法中。

  * 先让调用setNeedLayout和layoutIfNeeded来让cell完成布局，
  * 接着让自动布局计算systemLayoutSizeFittingSize:传入UILayoutFittingCompressedSize，意思是能用多小尺寸就用多小尺寸。

构建运行，然后你应当能看见有内容的table view了。

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/Populated-Table-View-RWBasicCell.jpg)

这看起来很棒！旋转屏幕，然后你就发现有有点奇怪：

![](http://cdn3.raywenderlich.com/wp-content/uploads/2014/05/Populated-Table-View-RWBasicCell-Landscape-Problem.jpg)

"为什么在subtitle标签附近有这么多空余呢？"你会问。

UILabel中有个preferredMaxLayoutWidth属性会影响它的布局。每当屏幕旋转时，preferredMaxLayoutWidth没有变。

幸运的是，你可以创建UILabel的子类(好复杂)，在项目中添加一个新的类，命名为RWLabel，使得它是UILabel的子类。

在RWLabel.m文件中，把@implementation里面的东西都换了

<pre class="brush: objc">

- (void)layoutSubviews {
[super layoutSubviews];

if (self.numberOfLines == 0) {

// If this is a multiline label, need to make sure 
// preferredMaxLayoutWidth always matches the frame width 
// (i.e. orientation change can mess this up)

if (self.preferredMaxLayoutWidth != self.frame.size.width) {
self.preferredMaxLayoutWidth = self.frame.size.width;
[self setNeedsUpdateConstraints];
}
}
}

- (CGSize)intrinsicContentSize {
CGSize size = [super intrinsicContentSize];

if (self.numberOfLines == 0) {

// There's a bug where intrinsic content size 
// may be 1 point too short

size.height += 1;
}

return size;
}

</pre>

如注释所写的，RWLabel解决了两个问题：

1. 如果Label有多行，它确保preferredMaxLayoutWidth这个值总是等于frame的宽度

2.有时候，intrinsicContentSize可能会太小。如果label有多行的话，intrinsicContentSize会增1.

现在，更新RWBasicCell类，在RWBasicCell.h文件里面，增加下列代码

<pre class="brush: objc">
#import "RWLabel.h"
</pre>

然后更新@property
<pre class="brush: objc">
@property (nonatomic, weak) IBOutlet RWLabel *titleLabel;
@property (nonatomic, weak) IBOutlet RWLabel *subtitleLabel;
</pre>

在Main.storyboard中，切换到Feed View Controller中，将title标签改成RWLabel类

![](http://cdn2.raywenderlich.com/wp-content/uploads/2014/05/RWBasicCell-Set-RWLabel-Class.jpg)

对subtitle标签也这么干。

在RWFeedViewController.m文件中，在calculateHeightForConfiguredSizingCell:方法中添加下列代码（第一行）

<pre class="brush: objc">
sizingCell.bounds = CGRectMake(0.0f, 0.0f, CGRectGetWidth(self.tableView.bounds), 0.0f);
</pre>

这会使得每个RWLabel更新自己的preferredMaxLayoutWidth属性。

构建、运行；无论横屏还是竖屏，label应该都能正常显示了。

![](http://cdn5.raywenderlich.com/wp-content/uploads/2014/05/Oh-Yeah.png)

### 图像呢？图像在哪儿

写不动了，不想写了。

<!-- create time: 2014-10-17 10:20:35  -->

