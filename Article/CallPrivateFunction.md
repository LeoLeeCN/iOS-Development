# **Swift 调用OC私有方法**

### 最近因为项目需求，需要调用WKWebview的私有方法。如果是用OC，那么有很多种方法。但是在StackOverFlow上搜了一圈，没发现有用的方法。于是就查资料研究一番。
###  OC 的函数是属于动态调用，在编译的时候是不能决定真正去调用那个函数的，只有在运行的时候才能决定去调用哪一个函数 ，在编译阶段，OC可以调用任何的函数，即使这个函数没有实现，只要声明过也就不会报错。
### Swift弱化了Runtime。纯Swift类的函数的调用已经不是OC的运行时发送消息，和C类似，在编译阶段就确定了调用哪一个函数，所以纯Swift的类我们是没办法通过运行时去获取到它的属性和方法的。
### 但是如果是继承自OC的Swift class， 还是可以用Runtime来调用私有方法和属性的。

## Swift 调用OC 私有方法
```Swift
let selector = NSSelectorFromString("_setObscuredInsets:")
method(UIEdgeInsets(top: 94, left: 0, bottom: 0, right: 0))

func extractMethodFrom2(owner: AnyObject, selector: Selector) -> ((Any?) -> Any)? {
    var temp = class_getMethodImplementation(WKWebView.self, selector)
        
    if let implementation = temp {
        typealias Function = @convention(c) (AnyObject, Selector, Any?) -> Void
            
        let function = unsafeBitCast(implementation, to: Function.self)
            
        return { userinfo in function(owner, selector, userinfo) }
            
    } else {
        return nil
    }
}
```

## Swift 调用OC 私有属性
```Swift
webView.setValue(true, forKey: "_haveSetObscuredInsets")
```

## 如何查看一个Swift class 的那些function和property继承自OC
```Swift
showClsRuntime(cls: WKBackForwardList.self)

    func showClsRuntime(cls:AnyClass) {
        
        print("function")
        
        var methodNum:UInt32 = 0
        
        let methodList = class_copyMethodList(cls, &methodNum)
        
        for index in 0..<numericCast(methodNum) {
            let method:Method = methodList![index]
            
            if let t1 = method_getTypeEncoding(method) {
                print(String(utf8String: t1) ?? " ",terminator: " ")
                print(String(utf8String: method_copyReturnType(method)) ?? " ",terminator: " ")
                print(String(_sel: method_getName(method)),terminator: " ")
                print("\n")
            }
        }
        
        print("property")
        var propertyNum:UInt32 = 0
        let propertyList = class_copyPropertyList(cls, &propertyNum)
        
        for index in 0..<numericCast(propertyNum) {
            let property:objc_property_t = propertyList![index]
            print(String(utf8String: property_getName(property)) ?? " ",terminator: " ")
            print(String(utf8String: property_getAttributes(property)!) ?? " ",terminator: " ")
            print("\n")
        }
    }
```