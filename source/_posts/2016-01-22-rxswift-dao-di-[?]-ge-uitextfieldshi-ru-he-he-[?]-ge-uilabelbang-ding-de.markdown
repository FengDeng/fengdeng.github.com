---
layout: post
title: "RxSwift之一个UITextField是如何和一个UILabel绑定的"
date: 2016-01-22 16:23:30 +0800
comments: true
categories: 
---


只要一行代码
	
	writeTextField.rx_text.bindTo(displayLabel.rx_text)

看下效果

![image](https://raw.githubusercontent.com/FengDeng/Blog_Image/master/RxSwift.gif)

那么这到底是如何实现的呢

<!--more-->

##整体分析
	代码层：
	writeTextField的rx_text通过bingTo这个函数和displayLabel的rx_text绑定了。
	
	Rx思想：
	一个被观察者(Observable)通过bingTo函数和一个观察者(Observer)绑定了，这样无论被观察这如何变化，观察者都了如指掌
	
##深层解析

###writeTextField.rx_text

这是一个Observable。也是一个Observer。
rx_text是RxSwift给UITextField的一个拓展属性，上代码

	public var rx_text: ControlProperty<String> {
        return rx_value(getter: { [weak self] in
            self?.text ?? ""
        }, setter: { [weak self] value in
            self?.text = value
        })
    }
    

rx_text是一个`ControlProperty<String>`。由后面的函数生成。我们去看下那个函数

	func rx_value<T: Equatable>(getter getter: () -> T, setter: T -> Void) -> ControlProperty<T> {
        let source: Observable<T> = Observable.create { [weak self] observer in
            guard let control = self else {
                observer.on(.Completed)
                return NopDisposable.instance
            }

            observer.on(.Next(getter()))

            let controlTarget = ControlTarget(control: control, controlEvents: [.AllEditingEvents, .ValueChanged]) { control in
                observer.on(.Next(getter()))
            }
            
            return AnonymousDisposable {
                controlTarget.dispose()
            }
        }
            .distinctUntilChanged()
            .takeUntil(rx_deallocated)
        
        return ControlProperty<T>(values: source, valueSink: AnyObserver { event in
            MainScheduler.ensureExecutingOnScheduler()

            switch event {
            case .Next(let value):
                setter(value)
            case .Error(let error):
                bindingErrorToInterface(error)
                break
            case .Completed:
                break
            }
        })
    }
    
这个函数是UIControl的，所有UIControl的所有子类都有哈。UITextField是UIControl的子类

函数有点长，我们一步一步分析，其实就两句


先说下第一句，source的生成过程。source是一个Observable<T>，即这里的Observable<String>

1. 首先获取self，获取不到直接completed
2. 获取到了，直接Next一个刚刚传进来的getter。这里是`self?.text ?? ""`，即流走一个Observable<String>
3. 生成一个`ControlTarget`，这个`ControlTarget`每当`ValueChanged`都会流走一个Observable<String>（这个`ControlTarget`有点多，下回讲）
4. 上面三步走完后 ，这里的是一个Observable<String>。对这个Observable<String>进行`distinctUntilChanged()`,这个函数说明，只有String改变才会继续往下流
5. `takeUntil(rx_deallocated)`,直到这个UITextField被释放
6. source结束，source的这个流程就是上面的5步

其实如果一个UITextField的rx_text作为一个Observable，上面的代码已经足够了。



为了实现UITextField也可以作为observer，于是有了下面的代码。

再说第二句，通过source，生成一个`ControlProperty<String>`。这个被观察者才是本函数要返回的类型。它既是一个Observer，也是一个Observable

1. 直接调用`ControlProperty<PropertyType>`的初始化方法生成
2. 看一下初始化方法带进去的一个闭包

		AnyObserver { event in
            MainScheduler.ensureExecutingOnScheduler()

            switch event {
            case .Next(let value):
                setter(value)
            case .Error(let error):
                bindingErrorToInterface(error)
                break
            case .Completed:
                break
            }
            
 
因为UI操作都必须在主线程。`MainScheduler.ensureExecutingOnScheduler()`这一句话是判断当前线程是不是在主线程           
下面的代码是当UITextField作为一个observer时，接收到一个流的响应。




###displayLabel.rx_text

这是一个Observer
rx_text是RxSwift给UILabel的一个拓展属性，上代码

	public var rx_text: AnyObserver<String> {
        return AnyObserver { [weak self] event in
            MainScheduler.ensureExecutingOnScheduler()
            
            switch event {
            case .Next(let value):
                self?.text = value
            case .Error(let error):
                bindingErrorToInterface(error)
                break
            case .Completed:
                break
            }
        }
    }

很简单，只有响应流的代码。有流过来，就`self?.text = value`

有同学可能有疑问了，UITextField可以被观察，也可以是观察者。UILabel只能是观察者，我要是想观察UILabel的text怎么办呢？？？

哈哈，easy，please try KVO

	displayLabel.rx_observe(String.self, "text").subscribe { (event) -> Void in
            switch event{
            case .Next(let m):
                print(m)
            case .Completed:
                print("")
            case .Error(let t):
                print(t)
            }
        }
 
KVO还是比较简单的，大家可以自己看看。或许下一次我写写


###bindTo
`bingTo`是Observable的拓展

	public func bindTo<O: ObserverType where O.E == E>(observer: O) -> Disposable {
        return self.subscribe(observer)
    }
    
内部调用的Observable的subscribe方法来和一个observer进行关联

这里就是调用了ControlProperty<String>的subscribe方法.

	public func subscribe<O : ObserverType where O.E == E>(observer: O) -> Disposable {
	    return _values.subscribe(observer)
    }

    
_values.subscribe(observer)调用了SubscribeOn这个类的subscribe方法，


SubscribeOn没实现subscribe方法，于是去他的父类Producer里去调用了

	override func subscribe<O : ObserverType where O.E == Element>(observer: O) -> Disposable {
        if !CurrentThreadScheduler.isScheduleRequired {
            return run(observer)
        }
        else {
            return CurrentThreadScheduler.instance.schedule(()) { _ in
                return self.run(observer)
            }
        }
    }

Producer又去它的子类SubscribeOn去调用了run方法

	override func run<O : ObserverType where O.E == Ob.E>(observer: O) -> Disposable {
        let sink = SubscribeOnSink(parent: self, observer: observer)
        sink.disposable = sink.run()
        return sink
    }
	
	
当UITextField的每一个值产生时，调用了下面这个方法，在`SubscribeOnSink`这个类中

	func on(event: Event<Element>) {
        forwardOn(event)
        
        if event.isStopEvent {
            self.dispose()
        }
    }
    

`forwardOn(event)`这个方法又去`SubscribeOnSink`这个类的父类中去调用了

	final func forwardOn(event: Event<O.E>) {
        if disposed {
            return
        }
        _observer.on(event)
    }
    
最终调到了`_observer.on(event)`这一句话。也是就UILabel的rx_text的`on`方法，`on(event)`是一个闭包，在UILabel的rx_text中等同以下代码
	
		{ [weak self] event in
            MainScheduler.ensureExecutingOnScheduler()
            
            switch event {
            case .Next(let value):
                self?.text = value
            case .Error(let error):
                bindingErrorToInterface(error)
                break
            case .Completed:
                break
            }
            
以上是一个复杂的继承🌲的方法调用。多看多揣摩吧！
            
###至此，一个完整的Observable和一个Observer的绑定结束


	

	


