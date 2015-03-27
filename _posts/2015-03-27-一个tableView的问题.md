---
layout: default
title: 一个tableView的问题
---

## 一个问题

我有这样如下需求：有一个tableView(继承自github上的SKSTableView)，其中cell有两种，一种是可扩展的cell，点开可以显示更多子列表；
另一种是可下载的cell，cell右侧显示了一个下载标志，点击可下载的cell，后台就会启动下载(基于AFHTTPRequestOperation)，并且cell
上还"覆盖"了一个progressView，能显示下载进度(也是通过AFNetworking的 setDownloadProgressBlock 实现的)。下载完成后那个progressView
会消失，并且tableViewCell也会"进化"成可扩展的cell。

有个小问题，我的添加progressView的操作是这么写的

```object-c

MYTableViewCell *cell = (MYTableViewCell*)[_tableView cellForRowAtIndexPath:aIndexPath];

MYProgressView *progressView = [[MYProgressView alloc] init];

//some init codes (settting progressView's frame and progress values)

[cell.contentView insertSubView:progressView atIndex:0];

[MYoperation setCompletionBlockWithSuccess:^(AFHTTPRequestOperation *operation, id responseObject) {
        NSLog(@"Successfully downloaded file");
        cell.expandable = YES;
        [progressView removeFromSuperview];
        progressView = nil;
    } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
        NSLog(@"Error: %@", error);
    }];

[MYoperation setDownloadProgressBlock:^(NSUInteger bytesRead, long long totalBytesRead, long long totalBytesExpectedToRead) {
        double percent = (double)totalBytesRead/totalBytesExpectedToRead;
        [progressView setProgress:percent animated:YES];
    }];

```
我想有经验的程序员能看出来会出什么问题，我点击了下载按钮，一定能开始下载，相应的cell上也会出现动画，不过，当用户scroll tableView，
cell变成invisible的时候，另外一个cell就莫名其妙也出现了下载动画，[这个问题](http://stackoverflow.com/questions/26216597/ios-uitableview-cell-changes-selection-state-when-scrolling)
应该能解释为什么会出现这种现象，我们所有的能改变cell外观的东西应该都写在dataSource里面，否则就会出现意想不到的情况。

```object-c
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
}
```

(不过我还没想出来怎么改代码！)
