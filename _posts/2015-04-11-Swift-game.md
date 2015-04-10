---
layout : default
title : Swift实现的一个小游戏
---

## ChessGame

用Swift实现了一个小游戏，比较简单，主要是想练习一下的protocol用法。

游戏截图

![](http://sae-gif.qiniudn.com/cardgame.png)

玩法很简单，马在棋盘上按“日”字走，你需要遍历这张棋盘(6*6是马能遍历的最小棋盘，你可以尝试写程序计算出遍历的方法，具体实现技巧可以[看这里](https://gilith.wordpress.com/2013/02/20/large-knights-tours/))。

## 实现

我定义了一个ChessView，它负责绘制黄橙相间的棋盘，并且当用户点击ChessView上的某一个点时，告诉controller用户到底点了哪一行、哪一列(通过delegate)

```swift

 protocol ChessViewDelegate  {
     func didTouchView(view:UIView,line:Int,row:Int)
}

class ChessView :UIView {
    
    var size:CGFloat = 0.0
    var delegate:ChessViewDelegate?
    
    override func drawRect(rect: CGRect) {
        size = min(rect.size.width, rect.size.height)
        size /= 6
        var context = UIGraphicsGetCurrentContext()
        let yellowColor = UIColor(red: 1.0, green: 0.896, blue: 0.545, alpha: 1.0).CGColor
        let orangeColor = UIColor(red: 0.991, green: 0.311, blue: 0.136, alpha: 1.0).CGColor
       
        var tapGesture = UITapGestureRecognizer(target:self, action: "handleGesture:")
        tapGesture.numberOfTapsRequired = 1
        tapGesture.numberOfTouchesRequired = 1
        self.addGestureRecognizer(tapGesture)
        
        for line in 0...5{
            for row in 0...5{
                if(line % 2 != 0){
                    if(row % 2 != 0){
                        CGContextSetFillColorWithColor(context, yellowColor)
                    }else{
                        CGContextSetFillColorWithColor(context, orangeColor)
                    }
                }else{
                    if(row % 2 != 0){
                        CGContextSetFillColorWithColor(context, orangeColor)
                    }else{
                        CGContextSetFillColorWithColor(context, yellowColor)
                    }
                }
                var aRect = CGRectMake(CGFloat(row) * size, CGFloat(line) * size, size, size)
                CGContextFillRect(context, aRect)
            }
        }
        CGContextFillPath(context)
    }
    
    func handleGesture(sender:UITapGestureRecognizer){
        let point:CGPoint = sender.locationInView(self)
        let line  = point.y/size
        let row = point.x/size
        delegate?.didTouchView(self, line: Int(line), row: Int(row))
    }
}

```

然后在Controller里面，实现<code>ChessViewDelegate</code>这个protocol，Swift的protocol和object-c里面有一个很大的区别，Swift的protocol默认状态下是@require的，也就是说你必须要实现protocol内的所有方法，否则编译器会报错。

Controller的实现太简单了，不好意思讲，代码写的实在不算满意,而且Swift的bug也很多，编译错误无法精确定位到错误的哪一行。

代码放在[github](https://github.com/RRanddom/ChessGame)上了.

啊！生活


<!-- create time: 2015-04-10 23:15:56  -->

<!-- This file is created from $MARBOO_HOME/.media/starts/default.md
本文件由 $MARBOO_HOME/.media/starts/default.md 复制而来 -->

