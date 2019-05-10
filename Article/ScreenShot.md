# **Get screenshot faster**

### Recently, I have focus on performance improvement in our project. After investigating by time profile, I find screenshot is another breakthrough that we can take action. It cost **100ms-300ms** on mainthread. 
<br/>

## **1. The old method**
### Following is the code used in our project before. It cost roughly 300ms on main thread, a big problem!
```Swift
UIGraphicsBeginImageContextWithOptions(viewBounds.size, false, 0)
view.drawHierarchy(in: viewBounds, afterScreenUpdates: false)
image = UIGraphicsGetImageFromCurrentImageContext()
UIGraphicsEndImageContext()
```
<br/>

## **So here have some questions:**

### 1. Is there a faster way to get screenshot?
### 2. Can I get screenshot on worker thread? Don't block main thread.
<br/>

## **2. A faster methord on iOS 10**
Actually we fond a good method, it's available after iOS 10. This methond only cost about 100ms. But not enough I think.

```Swift
let format = UIGraphicsImageRendererFormat()
let renderer = UIGraphicsImageRenderer(bounds: uiView.bounds, format: format)
let image = renderer.image { rendererContext in
    uiView.bounds.render(in: rendererContext.cgContext)
}
```

## **3. Run on background thread**
Can this new method run on background thread? Fortunately, the anwser is yes!!!<br/>
Apple development Doc has a mistake with this method. They told developers this method should run on main thread. But they retract the statement later.<br/>
The only think we need to do is prepare some properties(viewbound,layer) must got on main thread. 

```Swift

let viewBounds = uiView.bounds
let layer = uiView.layer

AsyncUtils.async {
    let format = UIGraphicsImageRendererFormat()
    let renderer = UIGraphicsImageRenderer(bounds: viewBounds, format: format)
    let image = renderer.image { rendererContext in
        layer.render(in: rendererContext.cgContext)
    }
}
```

<br/><br>
# **Not the End!!!**

## **What if must run on main thread?**
Sometimes we need get screenshot on main thread, such as get a screenshot for animation.
## So I use a simple if...else... to separate the OS version.

```Swift
 if #available(iOS 10.0, *) {
     //New method
 } else {
     //Old method
 }
 ```
<br/>

## **Hardware Support**
Our team has many kinds of iPhone, so I test those device one by one.<br/>
Then I found things are not that simple. Some old device(iPhone5) with higher OS(12.1.4) version support New method, but seems cost more cup tims. 
### After I test all the device with same OS version(12.1.4), I found that devices release later than iPhone SE, should use new method. Otherwise should use old method.

|  | Old method | New method |
| ------ | ------ | ------ |
| Device released later than iPhone SE | ~300ms | <100ms |
| Device released earlier than iPhone SE | >100ms | >300ms |


## So we need change our code as device model became a new consideration:

```Swift
if #available(iOS 10.0, *), UIDevice.modelName.ReleaseDate >= DeviceModel.iPhoneSE {
    //New method
} else {
    //Old method
}
```

### Thanks！！！
<br/>

### Donate

<img src="../Picture/AliPay.jpeg" width="200">