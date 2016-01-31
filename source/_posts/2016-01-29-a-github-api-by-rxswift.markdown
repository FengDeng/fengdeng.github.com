---
layout: post
title: "A GitHub API by RxSwift"
date: 2016-01-29 15:48:36 +0800
comments: true
categories: 
---


###一个用RxSwift实现的GitHubAPI库

仓库地址 <https://github.com/FengDeng/RxGitHubAPI>

这个库还未完善，由于最近工作比较忙，刚刚实现小部分的功能。

也希望大家pull request 来完善它

目前实现的功能：

1. 登录
2. 获取一个用户信息（自己的或者别人的）
3. 获取这个用户的其他信息，请看 [#YYUser]()
4. star，unStar,checkStar等等对仓库的一些行为，请看[#YYRepository]()
5. search Repos  和  search Users

<!--more-->


#Start
1. 设置 `RxGitHubUserName` `RxGitHubPassword`
2. 获取 login user 

		static var yy_user : Requestable<YYUser>		
3. 订阅`Requestable<YYUser>`就可以获取到你登录的用户了。


#Search
搜索仓库

	RxGitHubAPI.searchRepos(q:String）
	
订阅其获取YYRepository实例数组
	
搜索用户

	RxGitHubAPI.searchUsers(q:String）
	
订阅其获取YYUser实例数组



#YYUser

通过登录获取YYUser实例，先实现前面start方法

	RxGitHubAPI.yy_user
	
订阅其获取YYUser实例


通过username获取YYUser实例

	RxGitHubAPI.yy_user(username：）
	
订阅其获取YYUser实例

使用这个YYUser实例，获取更多信息
		
     	list followers     	
    	public var yy_followers:Pageable<[YYUser]
    
    
     	list followings
    	public var yy_followings : Pageable<[YYUser]>
    
    
     	list gists     
   	    public var yy_gist : Pageable<[YYGist]>
    
    
     	list starred repo
    	public var yy_starred : Pageable<[YYRepository]>
    
    
     	list subscriptions repo
    	public var yy_subscriptions : Pageable<[YYRepository]>
    
    
     	list subscriptions orgs
		public var yy_organizations : Pageable<[YYOrganization]>
    
    
     	list user repo
    	public var yy_repos : Pageable<[YYRepository]>
    
    
    	//fix me
    	//public var yy_events
    
    	//public var yy_received_events
    
    
    	//action needs Auth
    	
    	follow this user
      	public var action_follow : Actionable<Bool>
    
    
     	unFollow this user
		public var action_unFollow : Actionable<Bool>
    
    
     	check you are following this user
		public var action_checkFollow : Actionable<Bool>
		
		
#YYRepository

获取到YYRepository实例，你可以

		    /**
    star this repo
    
    - returns: return value description
    */
    public var action_star : Actionable<Bool>{
        return Actionable(mothod: .PUT, url: self.action_star_url.yy_clear(),headers:AuthHeader())
    }
    
    /**
     unStar this repo
     
     - returns: return value description
     */
    public var action_unStar : Actionable<Bool>{
        return Actionable(mothod: .DELETE, url: self.action_star_url.yy_clear(),headers:AuthHeader())
    }
    
    /**
     check you are starring this repo
     
     - returns:
     */
    public var action_checkStar : Actionable<Bool>{
        return Actionable(mothod: .GET, url: self.action_star_url.yy_clear(),headers:AuthHeader())
    }
    
    /**
     watch this repo
     
     - returns: return value description
     */
    public var action_watch : Actionable<Bool>{
        return Actionable(mothod: .PUT, url: self.action_watch_url.yy_clear(),headers:AuthHeader())
    }
    
    /**
     unWatch this repo
     
     - returns: return value description
     */
    public var action_unWatch : Actionable<Bool>{
        return Actionable(mothod: .DELETE, url: self.action_watch_url.yy_clear(),headers:AuthHeader())
    }
    
    /**
     check you are watching this repo
     
     - returns:
     */
    public var action_checkWatch : Actionable<Bool>{
        return Actionable(mothod: .GET, url: self.action_watch_url.yy_clear(),headers:AuthHeader())
    }


