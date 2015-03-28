---
layout: default
title: 一个tableView的问题
---

## 一个问题

我有这样的需求：有一个tableView(继承自github上的SKSTableView)，其中cell有两种，一种是可扩展的cell，点开可以显示更多子列表；
另一种是可下载的cell，cell右侧显示了一个下载标志，点击可下载的cell，后台就会启动下载(基于AFHTTPRequestOperation)，并且cell
上还"覆盖"了一个progressView，能显示下载进度(也是通过AFNetworking的 setDownloadProgressBlock 实现的)。下载完成后那个progressView
会消失，并且tableViewCell也会"进化"成可扩展的cell。

有个小问题，我的添加progressView的操作是这么写的

```objc

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

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
}
```

(不过我还没想出来怎么改代码！)

翻了一个多小时的stackoverflow，大概想明白了：iOS是 memory compact设备，所以tableView不应该持有_某一个_tableViewCell的信息，不应该通过上面讲的那种方法更新cell的外观，因为cell一旦消失在视野里，它在内存中的信息也就没有了，tableViewCell获得信息的渠道应该是dataSource。
所以应该把progressView这玩意儿放在MYTableViewCell里面，MYTableViewCell对外留一个progressValue的接口，progressValue不为0的时候...
欸，等等，下载怎么办！

## 解决方案(部分代码)

progressView 放在cell里面

```objc
\\cell.h
@property (nonatomic,strong) DCProgressView *progressView;
@property (nonatomic,assign,setter=setProgress:) double progress;
```

```
\\cell.m
- (void) setProgress:(double)progress
{
    self.progressView.progress = progress;
    if (progress > 0.0) {
        if (!_progressView) {
            self.progressView = [[DCProgressView alloc] initWithFrame:self.bounds];
            _progressView.tintColor = [UIColor colorWithRed:0.634 green:1.000 blue:0.538 alpha:0.800];
            _progressView.progress = 0.0;
            _progressView.rounding = 0.0;
            _progressView.alpha = 0.4;
            _progressView.fillColor = [UIColor colorWithRed:0.723 green:1.000 blue:0.997 alpha:1.000];
            [self.contentView addSubview:_progressView];
        }
    }
    
    if (fabs(progress-1.0)<0.0001) {
        [_progressView removeFromSuperview];
        _progressView = nil;
    }
}
```

```objc
\\tableViewController.m

- (void) downloadWithIndexPath:(NSIndexPath *)indexPath
{
    /*init the request and operation*/
    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:[NSString stringWithFormat:@"%@%d.zip",Downloadprefix,indexPath.row]]];
    AFHTTPRequestOperation *operation = [[AFHTTPRequestOperation alloc] initWithRequest:request];
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *path = [[paths objectAtIndex:0] stringByAppendingPathComponent:[NSString stringWithFormat:@"%d",indexPath.row]];
    operation.outputStream = [NSOutputStream outputStreamToFileAtPath:path append:NO];
    
    /*get the cell*/
    MYTableViewCell * cell = (MYTableViewCell *)[self.tableView cellForRowAtIndexPath:indexPath];
    
    [operation setCompletionBlockWithSuccess:^(AFHTTPRequestOperation *operation, id responseObject) {
        NSLog(@"Successfully downloaded file to %@", path);//
        \\cell  doFinishAnimation 
    } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
        NSLog(@"Error: %@", error);
        \\cell  doFailureAnimation
    }];
    
    [operation setDownloadProgressBlock:^(NSUInteger bytesRead, long long totalBytesRead, long long totalBytesExpectedToRead) {
        double percent = (double)totalBytesRead/totalBytesExpectedToRead;
        /*if cell is visible*/
        if ([[self.tableView indexPathsForVisibleRows] containsObject:indexPath])         {
            cell.progress = percent;
        }
        }];
    [operation start];
}
```

代码大概就是这样的，可能有些其他问题
