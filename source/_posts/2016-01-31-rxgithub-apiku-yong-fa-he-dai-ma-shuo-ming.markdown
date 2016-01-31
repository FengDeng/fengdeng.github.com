---
layout: post
title: "Swift，RxSwift实现的RxGithub API库用法和代码说明"
date: 2016-01-31 21:13:37 +0800
comments: true
categories: 
---

这是一个基于RxSwift的GitHub API库。希望给那些想写GitHub APP的同学一些方便。


#RxGitHubAPI的整体设计

所有的返回都设计成对象。
例如用户，YYUser
仓库，YYRepository
tag,YYTag
等等……

请大家移步去clone项目看吧 <https://github.com/FengDeng/RxGitHubAPI>

<!--more-->

所有的请求返回只有三种类型


1. Requestable<Element:NSObject>

2. Pageable<Element:_ArrayType where Element.Generator.Element : NSObject>

3. Actionable<Element:BooleanLiteralConvertible>

这三个全部都是符合Observable协议的类，也就是说它们是可以被订阅的。

下面来解释下这三个类型的主要设计用途
1. 怎么样获取一个用户？
2. 获取到了这个用户实例，怎么样获取他的所有仓库？
3. 获取到了所有仓库，怎么样给其中的一个仓库star？


###`Requestable<Element:NSObject>`
这是一个符合Observable协议的类，它是一个泛型类，且泛类型必须符合NSObject协议。之所有这样是因为本库JSON解析成模型用的是MJExtension。整个工程就这一个库是OC写的。也许有时间会换成Swift的解析库、

这个类主要是实现 请求 返回结果(错误或者正确)。
当你订阅这个类实例的时候，它就会去网络请求，并且返回结果你订阅者

具体例子，根据一个用户名获取这个用户的详细信息

	RxGitHubAPI.yy_user("tangqiaoboy").subscribe { (user) -> Void in
            switch user{
            case .Next(let user):
                print(user.login)
            case.Completed:
                print("completed")
            case .Error:
                print("error")
            }
        }
    //下面是控制台输出
    tangqiaoboy
	completed
	
	
完美！看看这个RxGitHubAPI.yy_user

	static func yy_user(username:String)->Requestable<YYUser>{
        return Requestable(mothod: .GET, url: kUserURL(username))
    }
        
短短几行代码。返回的是Requestable对象，泛型在这里是是YYUser。

YYUser就是RxGitHubAPI里表示一个用户信息的对象模型

###Pageable<Element: _ArrayType where Element.Generator.Element : NSObject>

这是一个符合Observable协议的类，它是一个泛型类，且泛类型必须符合_ArrayType协议，且成员必须符合NSObject协议。因为这是提供给那些返回数组的API使用的，所有它的泛型必须符合数组，这样才能给订阅者提供想要的类型。

这个类主要实现 请求  返回结果(错误或者正确)  翻页
当你订阅这个类的时候，他就会去网络请求，并且返回结果给订阅者。当你想翻页的时候，你可以对Pageable进行nextPage操作

具体例子，就拿上面那个例子来说。在上面我们不是获取到了一个tangqiaoboy用户的对象模型YYUser嘛。假设我们现在持有这个YYUser的实例，就叫tangQiaoUser.我们来看看怎么获取这个tangqiaoUser的所有public仓库。

	RxGitHubAPI.yy_user("tangqiaoboy").subscribe { (user) -> Void in
            switch user{
            case .Next(let user):
                print(user.login)
                //获取到了tangqiao用户
                //获取这个用户的所有公开仓库
                user.yy_repos.subscribeNext({ (repos) -> Void in
                    for repo in repos{
                        print(repo.full_name)
                    }
                })
            case.Completed:
                print("completed")
            case .Error:
                print("error")
            }
        }
        
     //控制台输出如下
    tangqiaoboy/AVFoundationDemo
	tangqiaoboy/ClassNote-iOS
	tangqiaoboy/DocSets-for-iOS
	tangqiaoboy/DoubanAlbum
	tangqiaoboy/ELCImageGrabber
	tangqiaoboy/ELCImagePickerController
	tangqiaoboy/FlurryUsageSample
	tangqiaoboy/FmdbSample
	tangqiaoboy/HashTest
	tangqiaoboy/Hello-World
	tangqiaoboy/iOS-Pro
	tangqiaoboy/iOS-QR-Code-Generator
	tangqiaoboy/iOS5ViewCtrlerSample
	tangqiaoboy/iOSBlogCN
	tangqiaoboy/iOSSF
	tangqiaoboy/iphone-app
	tangqiaoboy/iRuby
	tangqiaoboy/juzhai
	tangqiaoboy/KVO_Sample
	tangqiaoboy/LTBlacklist
	tangqiaoboy/MultiLayerNavigation
	tangqiaoboy/newsyc
	tangqiaoboy/octopress
	tangqiaoboy/quotation.github.io
	tangqiaoboy/Ready2Rock
	tangqiaoboy/spf13-vim
	tangqiaoboy/SynchronizedUIActionSheetDemo
	tangqiaoboy/tangqiaoboy.github.com
	tangqiaoboy/the-swift-programming-language-in-chinese
	tangqiaoboy/TopWindowSample

是不是超简单？？

这里默认的是第一页，每页三十条，当然了，你可以通过下面的方法给改变页数，或者每页的条数。

	//获取到了tangqiao用户
                //获取这个用户的所有公开仓库
                user.yy_repos.page(2, per_page: 1).subscribeNext({ (repos) -> Void in
                    for repo in repos{
                        print(repo.full_name)
                    }
                })
                
简单到令人发指！！！！



###Actionable<Element:BooleanLiteralConvertible>

这个类其实就是为了处理那些返回Bool的请求。

直接上例子吧。还是接着上面的。现在我有了tangqiaoboy的仓库模型YYRepository的实例，我想star。怎么star？？

下面对刚刚获取的仓库列表的第一个进行star！简单：

	RxGitHubAPI.yy_user("tangqiaoboy").subscribe { (user) -> Void in
            switch user{
            case .Next(let user):
                print(user.login)
                //获取到了tangqiao用户
                //获取这个用户的所有公开仓库
                user.yy_repos.page(2, per_page: 1).subscribeNext({ (repos) -> Void in
                    repos[0].action_star.subscribeNext({ (isSuccess) -> Void in
                        if isSuccess{
                            print("您关注成功啦！")
                        }
                    })
                })
            case.Completed:
                print("completed")
            case .Error:
                print("error")
            }
        }


是不是很简单方便？？还有很多工作为完成，后面的开发估计要等到年后了。大家没事就手贱star吧！给我点动力哈！！

欢迎大家去GitHub查看[源代码](https://github.com/FengDeng/RxGitHubAPI)

也欢迎大家去[新浪微博](http://weibo.com/FengDeng1219)去给我发私信

感谢大家！