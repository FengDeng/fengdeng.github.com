---
layout: post
title: "RxSwift Observable之Empty,Map的实现"
date: 2016-01-13 09:32:34 +0800
comments: true
categories: 
---

上一节说了[Observable这个超类的设计](http://fengdeng.github.io/blog/2016/01/12/rxswiftxi-lie-zhi-observable/)，本节内容来浅析下它的众多子类中的两个的实现

##Empty
先看下一个sample

	example("empty") {
    let emptySequence = Observable<Int>.empty()

    let subscription = emptySequence
        .subscribe { event in
            print(event)
        }
	}
	
	//输出
	--- empty example ---
	Completed
	
Observable<Int>初始化一个类型为Int的Observable实例，来看看empty()方法

	public static func empty() -> Observable<E> {
        return Empty<E>()
    }
    
<!--more-->
    
在本例中其实是实例化了一个Empty<Int>,我们去看看Empty的实现

	class Empty<Element> : Producer<Element> {
    	override func subscribe<O : ObserverType where O.E == Element>(observer: O) -> Disposable 	{
        	observer.on(.Completed)
        	return NopDisposable.instance
    	}
	}
	
看到了没，是Producer的子类，Producer是啥，是Observable的子类，说明这是一个可被观察的对象。它重写了订阅方法。

暂且我们没有写到`Observer`，但是这里我先说明，这里的observer.on(.Completed)里的`on(.Completed)`等于我们sample里的`{ event in print(event)}`。

这说明啥，说明我们订阅这个Empty的实例，其实内部就是对我们的闭包进行了一次回调，传出`.Completed`的Event。

这样应该就明白为什么会输出`//输出
	--- empty example ---
	Completed`

可能有人会疑问，那么Empty里的observer.on(.Completed)里的observer怎么来的，sample里根本没有传observer啊。
我们看sample里的subscribe方法，在ObservableType协议里有声明

	public func subscribe(on: (event: RxSwift.Event<Self.E>) -> Void) -> Disposable
	
其实就是传入一个闭包，根本没有Observer.那么sample是怎么调用Empty里的订阅方法的呢？

我们来看`ObservableType`的一个拓展，
	
	public func subscribe(on: (event: Event<E>) -> Void)
        -> Disposable {
        let observer = AnonymousObserver { e in
            on(event: e)
        }
        return self.subscribeSafe(observer)
    }
    
原来这样子啊。Empty找不到sample里的subscribe方法，只能到父类里去找了，最终找到了在协议的默认实现里找到了，于是调用。

这个方法创建了一个匿名观察者，然后self.subscribeSafe(observer)，

	func subscribeSafe<O: ObserverType where O.E == E>(observer: O) -> Disposable {
        return self.asObservable().subscribe(observer)
    }
    
这里的subscribe(observer)就是调用的Empty里的订阅方法。observer就来了。并且observer还把我们的闭包带来了。
于是打印`Completed`

哈哈

以上整个就是订阅一个空信号的过程。

##举一反三，兄弟们自己对照源码看看Never Just等等同类型的实现

##Map

Map，就是一个关卡，流过这个关卡的Observable都会被这个关卡所改变。


看下Map的sample

	example("map") {
    let originalSequence = Observable.of(Character("A"), Character("B"), Character("C"))

    _ = originalSequence
        .map { char in
            char.hashValue
        }
        .subscribe { print($0) }
	}
	
这个sample是对一个队列进行map，闭包里把char转换为hashValue。

originalSequence是一个被观察者，看看这个map方法的定义和实现

	public func map<R>(selector: E throws -> R)
        -> Observable<R> {
        return self.asObservable().composeMap(selector)
    }
    
传入一个E->R的闭包，返回一个Observable<R>，看一下self.asObservable().composeMap(selector)

	internal func composeMap<R>(selector: Element throws -> R) -> Observable<R> {
        return Map(source: self, selector: selector)
    }
    
    
原来返回的是一个Map的实例。。。而这个Map也是继承Producer，所以也是Observable<R>
 
	class Map<SourceType, ResultType>: Producer<ResultType> 
 	
 
当这个Map被订阅的时候，它的run方法被调用，

	override func run<O: ObserverType where O.E == ResultType>(observer: O) -> Disposable {
        let sink = MapSink(selector: _selector, observer: observer)
        sink.disposable = _source.subscribe(sink)
        return sink
    }
    
MapSink是个什么鬼。。。翻阅源码就会发现，所有涉及到信号的变化的observable，都会有一个sink。这个sink是为了多线程而设计的。里面大多数都是原子操作，自旋锁，容错机制。

sample的原始信号是一个Sequeue,在sequence的实现里可以知道，对sequeue进行循环直到completed。这里对一次循环进行说明。

一次循环，调用一次MapSink(selector: _selector, observer: observer)。

MapSink里调用一次这个方法

	func on(event: Event<SourceType>) {
        switch event {
        case .Next(let element):
            do {
                let mappedElement = try _selector(element, try incrementChecked(&_index))
                forwardOn(.Next(mappedElement))
            }
            catch let e {
                forwardOn(.Error(e))
                dispose()
            }
        case .Error(let error):
            forwardOn(.Error(error))
            dispose()
        case .Completed:
            forwardOn(.Completed)
            dispose()
        }
    }
    

通过这一句话`let mappedElement = try _selector(element, try incrementChecked(&_index))`这个整个流就从`.Next(E)->.Next(R)`

这个流程如下

`数组<E>------>被观察者<E>------->被观察者<Map<E->R>>`
	

##举一反三，兄弟们自己对照源码看看同类型的实现


##一些总结


RxSwift中四个重要的角色

`Observable`

`Observer`  

`Disposable`

`Scheduler`

研究好了这个四个鬼，就研究好了Rx。


下一节详解一个Model的text绑定一个UITextField的text的响应流程





