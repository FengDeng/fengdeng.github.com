---
layout: post
title: "RxSwift思想"
date: 2016-01-19 17:18:45 +0800
comments: true
categories: 
---

##流的世界——初识

想象一下你的手机屏幕下面是一个管道，这个管道的入口连接在屏幕上。

我们的每一次点击，滑动都是向这个管道下发一个东西Observable<东西>。

		//举个例子 一次下载按钮的点击
		Observable<> --> Observable<>
		





