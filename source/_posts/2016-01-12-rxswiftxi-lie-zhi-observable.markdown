---
layout: post
title: "RxSwift系列之Observable"
date: 2016-01-12 19:10:09 +0800
comments: true
categories: 
---




流，信号，阀门，水管，水珠，漏斗，等等。这些名词都被大家用到RxSwift上，应该都了解，这里就不详细说了。
下面简单的梳理下RxSwift中的半壁江山，`Observable`

`Observable` 是一个泛型类

	public class Observable<Element> : ObservableType
	
其实现了`ObservableType`协议

	public protocol ObservableType : ObservableConvertibleType{
		typealias E	
		....
	}
	
`ObservableType`是一个泛型协议
实现这个协议的类，必须是可被观察的，所以`ObservableType`定义了一个订阅方法

	func subscribe<O: ObserverType where O.E == E>(observer: O) -> Disposable

`ObservableType`继承了`ObservableConvertibleType`协议，也是一个泛型协议。
从字面意思很容易看出来，只要实现了这个协议的类必须实现一个能使自己成为`Observable`的方法，所以`ObservableConvertibleType`定义了一个asObservable的方法

	func asObservable() -> Observable<E>	

来看`Observable`的实现，他实现了上面所说的两个协议的方法

	public func subscribe<O: ObserverType where O.E == E>(observer: O) -> Disposable {
        abstractMethod()
    }
    
    public func asObservable() -> Observable<E> {
        return self
    }
    
第一个方法是一个虚方法，提供给子类进行重写。如果子类不重写，直接抛出异常。我们来看看`abstractMethod()`

	@noreturn func abstractMethod() -> Void {
	    rxFatalError("Abstract method")
	}

	@noreturn func rxFatalError(lastMessage: String) {
    // The temptation to comment this line is great, but please don't, it's for your own 		good. The choice is yours.
    	fatalError(lastMessage)
	}

第二个方法直接返回自己。


接下来我们看看`Producer`这个类，它继承自`Observable`

	class Producer<Element> : Observable<Element>
	
并且具体的实现了订阅这个方法

	override func subscribe<O : ObserverType where O.E == Element>(observer: O) ->Disposable {
        if !CurrentThreadScheduler.isScheduleRequired {
            return run(observer)
        }
        else {
            return CurrentThreadScheduler.instance.schedule(()) { _ in
                return self.run(observer)
            }
        }
    }
    
    func run<O : ObserverType where O.E == Element>(observer: O) -> Disposable {
        abstractMethod()
    }

在订阅里调用了`run`方法。很明显，这个方法也是一个虚方法，子类必须重写.

接下来，我们可以预见，所有的过滤器其实就是继承`Producer`，并且重写`run`方法。这里的过滤器指`Empty``Map``Never`等待。

下一节再具体说说他们各自是怎么样实现`run`的

