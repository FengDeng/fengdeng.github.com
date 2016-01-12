---
layout: post
title: "CocoaAsyncSocket源码解读"
date: 2015-11-20 16:10:24 +0800
comments: true
categories: 
---

###先看看文件结构：
		├── CocoaAsyncSocket.h
		├── GCD
		│   ├── GCDAsyncSocket.h
		│   ├── GCDAsyncSocket.m
		│   ├── GCDAsyncUdpSocket.h
		│   └── GCDAsyncUdpSocket.m
		└── RunLoop
    		├── AsyncSocket.h
    		├── AsyncSocket.m
    		├── AsyncUdpSocket.h
    		└── AsyncUdpSocket.m
  一共9个文件
  
  CocoaAsyncSocket.h没什么说的，只是import了所有的头文件。
  
  根据作者的目录很容易可以看出来，作者提供了两套异步socket接口，一套基于`GCD`，一套基于`RunLoop`。
  
##基于RunLoop的socket通信
###先简单说一下AsyncSockt的工作流程：

1. 创建一个socket

		- (id)init;
		- (id)initWithDelegate:(id)delegate;
		- (id)initWithDelegate:(id)delegate userData:(long)userData;
		
2. 监听一个端口，如果是一个服务端，则向其读取信息。如果是一个客户端则向其发送信息
		- (BOOL)connectToHost:(NSString *)hostname
			   onPort:(UInt16)port
		  withTimeout:(NSTimeInterval)timeout
				error:(NSError **)errPtr;
3. 根据delegate进行回调，这里就不细写了，头文件都有

###首先来看AsyncSocket这两个文件
AsyncSocket这个类的属性

		@interface AsyncSocket : NSObject
		{
			CFSocketNativeHandle theNativeSocket4;
			CFSocketNativeHandle theNativeSocket6;
			
			CFSocketRef theSocket4;            // IPv4 accept or connect socket
			CFSocketRef theSocket6;            // IPv6 accept or connect socket
			
			CFReadStreamRef theReadStream;
			CFWriteStreamRef theWriteStream;
		
			CFRunLoopSourceRef theSource4;     // For theSocket4
			CFRunLoopSourceRef theSource6;     // For theSocket6
			
			//自己的runloop
			CFRunLoopRef theRunLoop;
			
			CFSocketContext theContext;
			
			NSArray *theRunLoopModes;
			
			//超时定时器
			NSTimer *theConnectTimer;
			
			//一个队列，存放服务端返回的信息
			NSMutableArray *theReadQueue;
			
			//如果信息被读走，且theReadQueue不为空，则从theReadQueue取出第一个包
			AsyncReadPacket *theCurrentRead;
			
			//一个读取buffer的定时器
			NSTimer *theReadTimer;
			
			
			NSMutableData *partialReadBuffer;
			
			NSMutableArray *theWriteQueue;
			AsyncWritePacket *theCurrentWrite;
			NSTimer *theWriteTimer;
		
			id theDelegate;
			UInt16 theFlags;
			
			long theUserData;
		}

