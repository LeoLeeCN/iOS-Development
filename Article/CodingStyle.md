# **Tips of Coding Style**

**Good habits make a good life.**

### After focus on performance improement for few days. I'd like to raise some advice here. You might think it's simple, but it's still deserve review.

## **1. Don't repeat**
When user swipe on webview, Edge will auto show/hide address bar. Here we need to get address bar's height.

Here has an example. This property is the height of address bar in browser. The address bar will auto show/hide when user swipe on webview.<br>
A simple property, doesn't seem to take long time. But cumulative time is huge, we need to refine it.

```Swift
static var height: CGFloat {
        return HeaderViewController.Heights.searchBoxHeight
            + (SizeClassManager.shared.isSizeClassLarge ? 0 : 2 * HeaderViewController.Heights.stackViewTopBottomPadding)
            + (SizeClassManager.shared.isSizeClassLarge ? 10 : 0)
            + HeaderViewController.Heights.separatorLineHeight
            + (SizeClassManager.shared.isSizeClassLarge ? HeaderViewController.Heights.tabBarHeight : 0)
    }
```

It's simple calculation, right? <br/>
But it will calculate Whenever user swipe. Then it might affect UI Thread.

So I change it as following, only calculate once.

```Swift
static var height: CGFloat {
        return SizeClassManager.shared.isSizeClassLarge ? HeaderViewController.largeSizeHeight : HeaderViewController.normalSizeHeight
    }

    static let largeSizeHeight: CGFloat = HeaderViewController.Heights.searchBoxHeight
            + HeaderViewController.Heights.separatorLineHeight
            + HeaderViewController.Heights.tabBarHeight
            + 10
    
    static let normalSizeHeight: CGFloat = HeaderViewController.Heights.searchBoxHeight
            + HeaderViewController.Heights.separatorLineHeight
            + 2 * HeaderViewController.Heights.stackViewTopBottomPadding
```

## **2. Don’t belittle a single statement**
One single line maybe cost more than 100ms on UI thread. That would be a disaster if run on UI Thread.
Such as:<br/>
<img src="../Picture/Performance/NSLinguisticTagger.png" width="600">

**The best solution is post on worker thread**


# **To be continued**

<img src="../Picture/AliPay.jpeg" width="200">
