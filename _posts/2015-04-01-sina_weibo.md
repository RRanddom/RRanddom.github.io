---
layout: default
title: weibo tools
---

这篇文章是以前写的 (zjcblog.sinaapp.com)

# 一个微博分析工具

最近刷微博的时候突然想写一个微博分析工具，主要分析一篇微博的转发行为，顺便发现一些有趣的现象。这个工具我写了一半，目前已经（勉强）可以用了，让我们看一下当红偶像，某boys成员的一条微博下的评论分析。

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](http://sae-gif.qiniudn.com/gender.png)

这是评论者性别分布（一目了然，哈哈哈）

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](http://sae-gif.qiniudn.com/heatmap.png)

这是评论者地域分布（这方面功能还不太完善，所以看起来有点不直观，地图上文字还没换成中文，o(╯□╰)o），可以看出评论者大多居住在北京和广州。

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](http://sae-gif.qiniudn.com/create_at.png)

这是评论者微博创建时间分布（嗯，还是一目了然），绝大多数评论者都是在2014年创建账号的。

我还希望加的功能是评论者年龄分布、评论者微博热度（发微博频度、微博粉丝数和关注数）、评论时间分布（观察是否有短时间内大量评论的现象）。当然，这些功能没有啥了不起的意义，它主要功能是娱乐。那些微博红人可以用这个工具观察自己微博的评论情况，闲杂人等也可以分析一下热门微博。

这个工具主要就是调用了[新浪微博的api](http://open.weibo.com/wiki/)，热点图用的是iOS自带的MapKit。

我下面介绍一下新浪微博API的使用吧，我觉的非常有意思。我们主要用到的API名字叫做[comments/show](http://open.weibo.com/wiki/2/comments/show)。就是评论读取的接口，要使用它的话，你先得有一个Acess Token（随便注册个应用就行啦）和一条微博的ID。向 _https://api.weibo.com/2/comments/show.json_ 发送一个HTTP GET Request，如果参数正确，一会儿，服务器就会返回一个json file，大概就是下面的结构
<code>
	
	jsonFile := Comments marks hasvisible previous_cursor total_number
	Comments := comment的集合
	total_number := 总评论数
	comment := created_at id text source user ....(blabla)
	user := id screen_name name province city description created_at ...(blabla)
	create_at := 用户创建时间
	text := 评论内容
	...
</code>

总之我们感兴趣的就是Comment Array，这是分析的主体，单条评论里面还有评论发送人(user)的全面信息，从user中我们可以提取出性别，地点，建号时间，粉丝数，关注数之类的。对了，[新浪微博api页面](http://open.weibo.com/wiki/2/statuses/public_timeline)还有一个bug，[API测试工具]()没法响应点击事件，换了几个浏览器都解决不了这个问题，只能在页面源码里面点链接。

通过这个API获得的是某条微博的第一页的50条评论，我们希望获得所有评论数，comments/show 还提供了page参数，默认是1，你可以通过“翻页”获得第二页、第三页的评论，只要在Http Request中更换page参数就好啦。让我们想一下在代码里怎么写，Cocoa的HTTP request分为同步和异步两种（这是废话），同步请求阻塞主线程，也就是UI线程，直到服务器返回数据。这是很不科学的，因为我们API返回的数据太大了(我测试了一下，50条评论大概有231KB大)，处理两千条评论就相当于下载10M的文件，所以我们必须在其他线程里面进行数据请求的操作，也就是异步Http Request。

Cocoa Touch中，你要Make Http Request，需要初始化一个NSURLConnection。如果是同步请求，直接sendSynchronousRequest，然后等待方法返回值就行了；如果是异步的请求，自然是sendAsynchronousRequest，然后通过实现代理方法来接受数据。这是“自动”的异步请求，你也可以自己开辟一个线程，然后在里面send Synchronous Request，效果是一样的。开辟线程的方法也有不少，最原始的应该就是pthread，那是C时代的多线程API，比较难用。Cocoa对其封装了一下，叫做NSThread，不过还是不够好用。我决定使用神器GCD(Grand Central Dispatch)，它能让你使用多线程的同时完全感受不到多线程的存在，至于GCD的原理：线程池，block之类的，我准备有时间研究一下，这里只讲简单使用。

我们的程序逻辑大概是是这样的，开辟另外一个线程，通过Http Request获取微博第一页的评论(50条)，从返回的json data中获取该微博评论总数，然后决定翻页次数。代码我是这么写的

```objc
	
	dispatch_async(dispatch_get_global_queue(0,0), ^{
	
		dispatch_sync(dispatch_get_global_queue(0,0), ^
		NSURLRequest *request = [[NSURLRequest alloc] initWithURL:url
		                                              cachePolicy:NSURLRequestUseProtocolCachePolicy
													  timeoutInterval:15];
		NSData *received = [NSURLConnection sendSynchronousRequest:request
		                       returningResponse:nil error:nil];
		id jsonFormatComments = [NSJSONSerialization JSONObjectWithData:received options:NSJSONReadingAllowFragments error:nil];
		totol_comment = [[jsonFormatComments objectForKey:@"total_number"] integerValue];
		[Comment_Array addObjectsFromArray:[jsonFormatComment objectForKey:@"comments"]];
		
		NSUInteger paging;
		if (total_number >= 2000) {
		            paging = 40;  //max comments we can get
		        } else{
		            paging = (total_number/50)*50 < total_number ? 1 + total_number/50 : total_number/50;
		        }
		for	(int pageNum = 2;pageNum <= paging; pageNum++)
		{
			NSString *urlString = [NSString stringWithFormat:@"%@?id=%@&access_token=%@&comment=50&page=%d",Comment_Api_Url,Weibo_ID,Access_tokens,pageNum];
			NSURL *url = [NSURL URLWithString:urlString];
			NSURLRequest *request = [[NSURLRequest alloc] initWithURL:url
			                                                          cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:15];
			NSData *received = [NSURLConnection sendSynchronousRequest:request returningResponse:nil error:nil];
			id jsonFormatComment = [NSJSONSerialization JSONObjectWithData:commentListData options:NSJSONReadingAllowFragments error:nil];
			[Comment_Array addObjectsFromArray:[jsonFormatComment objectForKey:@"comments"]];
		}
		)};
		
		//返回主线程，让用户知道已经下载完数据。
		dispatch_async(dispatch_get_main_queue(), ^
		                       {
		                           [self.buttonField setEnabled:YES];
		                       });
	
	)
```

NSJSONSerialization的作用就是把json数据 parse成objc中基本的一些类型，比如Dictionary，NSArray，整数，字符串。获取到了评论数组，剩下来的工作就比较轻松了，男女比例什么的太简单了，只要注意饼图画得美观一些就行了，折线图也很简单。主要是地域热点图，我差不多想了两天。一开始我的想法很单纯，以城市作为最小的染色单位：假如评论者很多都来自于北京宣武区，那么地图上宣武区就是一块格外显眼的色块，只有极少人来自于呼伦贝尔，那么地图上呼伦贝尔就是一块很淡的色块，如果一个区域没有人，那就是白色（不染色）。所有染色用的色彩都是同一色系，最终的显示效果就会明暗分明但一点都不突兀。有点类似人口密度图那样。

![](http://sae-gif.qiniudn.com/human.jpg)

不过没有找到合适的API，百度地图有个热力图api，我调试了20分钟的Demo，都没有调通，卒。在github上[随便找了一个](https://github.com/ryanolsonk/HeatMapDemo.git)，下完就能用，而且我还知道它是怎么工作的.

接下来的工作就乏善可陈了，我找的热力图API接受一组位置信息（经纬度)。我从CommentArray中能获取一个User_Array，每个User都有一个Province和City属性，都是int类型，可以映射到具体的城市(见[这张表](http://api.t.sina.com.cn/provinces.xml))。我要做的是为每个具体的城市添加latitude 和 longitude属性。

其实这个省市映射表也是有问题的，广西的南宁+柳州重复了两次，重庆的县市特别的多（多的有些不正常了）。而且城市的ID是不连续的，我也不知道是为什么。具体映射的时候问题更大，很多人微博信息中的cityID=100，而100在表中根本就没有，我就只能当做cityID=1来处理。这是我通过半自动的方法增加了latitude和longitude之后的[表](http://sae-gif.qiniudn.com/location.xml)，希望能让其他开发者少花一些时间(我搞了差不多3个小时)。

其他的问题以后再更新.

<!-- create time: 2014-10-11 01:04:04  -->
