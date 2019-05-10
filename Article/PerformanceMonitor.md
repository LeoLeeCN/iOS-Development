# **线上卡顿监测： 用Runloop还是Ping？**

### 最近因为项目需求，做了一个线上卡顿监测。 查了很多资料，有人说用Runloop。也有zixun大神的 [GodEyes](https://github.com/zixun/GodEye)
### 但是经过测试，我发现这两种方法各有缺陷。下边将先介绍两种方式的优缺点，并给出自己的解决方案。 如有不对的地方，请各位大神指出。

## **什么是卡顿**
### 卡顿就是主线程没有足够的时间片去绘制UI，导致用户看到的UI动画不是连续的。比如scrollview在滚动的时候跳动。
### 卡顿在code层面有氛围两种：长时间大卡顿，连续小卡顿。
### 参考了微信及美团的方案，我把单次150ms作为大卡顿的阈值。五次50ms的卡顿作为连续小卡顿的阈值。


## **Runloop**
###假设大家都已经了解Runloop， 不了的的请自行Google。

### **原理**
### 在runLoopObserver中可以监测到以下几种状态：beforeTimers（2），beforeSources（4）， beforeWaiting（32）， afterWaiting（64）. 一般情况下是(2)-(4)-(32)-(64)-（2）循环。所以可以通过注册一个runloop的observer来进行卡顿监测。

### **代码实现**
```Swift
func startShortSerialLagMonitor() {
        AsyncUtils.async {
            self.semaphore = DispatchSemaphore(value: 0)
            self.runLoopObserver = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, CFRunLoopActivity.allActivities.rawValue, true, 0, {
                (observer: CFRunLoopObserver!, activity: CFRunLoopActivity) -> Void in
                self.activity = activity
                self.semaphore.signal()
            })
            
            CFRunLoopAddObserver(CFRunLoopGetMain(), self.runLoopObserver, CFRunLoopMode.commonModes)
            
            while true {
                let st: DispatchTimeoutResult = self.semaphore.wait(timeout: .now() + DispatchTimeInterval.milliseconds(100))
                if st == DispatchTimeoutResult.timedOut {
                    if self.runLoopObserver == nil {
                        self.timeoutCount = 0
                        return
                    }
                    if self.activity == CFRunLoopActivity.beforeSources || self.activity == CFRunLoopActivity.afterWaiting {
                        self.timeoutCount = self.timeoutCount + 1
                        if self.timeoutCount < 5 {
                            continue
                        }

                        Logger.info("Serial Laggy: 250 ms")
                    }
                }
                
                self.timeoutCount = 0
            }
        }
    }
```
### **缺点**
**划重点，以下必考**
### 一般来说，runloop处理事件时间主要出在两个阶段：beforeSources和beforeWaiting之间，afterWaiting之后。所以上边的代码监测beforeSources 和 afterWaiting这两个状态下是否超时是没问题的。
### 但是经过写代码测试，发现并不是这么简单。大家看下边的Log：
```Swift
RunloopId : 27, Activity : 2
RunloopId : 27, Activity : 4
RunloopId : 27, Activity : 32
begin of task
end of task
RunloopId : 27, Activity : 64
```
### 当我把一些耗时操作写在viewDidLoad或者viewWillAppear的时候，奇怪的事情发生了。
### **状态32是beforeWaiting，也就是说如果卡顿出现在viewDidLoad或者viewWillAppear中的时候，Runloop并不能及时通知observer当前的状态。也就是说监测不到卡顿。

## **Ping**
### Ping 这种方法是我在GodEyes中学习到的。这种方法不依赖Runloop的状态。是在另一个线程中向主线程push task。如果超过一定时间这个task没有被执行，就认为主线程卡顿。

### **代码实现**
```Swift
    func startLongSingleLagMonitor() {
        AsyncUtils.asyncOnSerialQueue(PerformanceManager.singleLagQueue) {
            while true {
                var isTimeout = true
                    
                AsyncUtils.asyncOnMainQueue {
                    isTimeout = false
                }
                    
                Thread.sleep(forTimeInterval: 150)
                    
                if isTimeout {
                    Logger.info("Long Laggy: 150 ms")
                }
            }
        }
    }
```
### **缺点**
### 当runloop队列中积累了很多task时，这种方法会造成误报。因为每个小task并没有超时。
### 相较于runloop，这种方法也监测不到连续的小卡顿。

## **最终方案**
### 比较了两种方法的优缺点，我在monitor中同时使用。并且改进Ping的方式。

### **代码实现**
### runloop的方式不变
### ping的方式改为，只有状态是beforeWaiting的时候才ping主线程。
### **代码实现**
```Swift
        func startLongSingleLagMonitor() {
        AsyncUtils.asyncOnSerialQueue(PerformanceManager.singleLagQueue) {
            while true {
                if self.activity == CFRunLoopActivity.beforeWaiting {
                    var isTimeout = true
                    let semaphore = DispatchSemaphore(value: 0)
                    
                    AsyncUtils.asyncOnMainQueue {
                        isTimeout = false
                        semaphore.signal()
                    }
                    
                    Thread.sleep(forTimeInterval: 150)
                    
                    if isTimeout {
                        Logger.info("Long Laggy: 150 ms")
                    }
                    _ = semaphore.wait()
                }
            }
        }
    }
```


<img src="../Picture/AliPay.jpeg" width="200">
