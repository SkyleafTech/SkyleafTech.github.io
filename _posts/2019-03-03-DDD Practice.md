---
layout:     post
title:      企業及應用架構IX
subtitle:   DDD Practice
date:       2019-02-16
author:     BY Skyleaf
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - 企業及應用架構
---
# 前言

> 這篇稍微紀錄一下這次讀本書第九篇的感想，這篇偏向實際程式碼的紀錄，由於看完前面這麼多理論，實際上會變成怎樣還是有點不踏實的感覺，大陸人講不踏實我覺得鰻貼切的，感覺虛虛的。

![图](https://images.unsplash.com/photo-1542376855-bc7e1d174ee1?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=80)

## Agenda

- NAA4E 專案


# NAA4E 專案

新版的專案已經做到ASP.NET Core 2.2 的版本了，但這不打緊，重點是DDD的概念學會了，再將專案導入ASP.NET Core也不遲，而且3.0還未正式上線，相信到了3.0才會真正的穩定，該修正的也都修完了。這專案先從Layer開始區分Domain Layer，Infrastructure Layer，Site。Sit裡面包含了Presentation Layer跟Application Layer。看到這裡我就想到我先前的拆分就不是正規的感覺，肯定是有其優點才這樣拆，因為多半我是將application要做的放到Controller內，而bus logic還是在Domain Layer裡面。目前我還沒有明確理由說這樣一定不好，但初學DDD，就該依照他的方式先做過後，等熟悉了在使用自己的一套方式。
![图](https://i.imgur.com/IieZmTF.png)

## Site - Presentation Layer
 
- 這是一個MVC專案，裡面包含基本MVC Route跟View跟Controller。
- 參考: IBuyStuff.Application，IBuyStuff.Domain
- 需要特別講的是Common這個資料夾，裡面有Device，Helpers，Identity，Enum。
  - Device跟Enum都是Extension，他只是以功能區分，我猜他只用在前台呈現部分所以他沒放到Infrastructure層內
  - Identity資料夾內都是會員機制驗證相關，類似我的JWT資料夾
  - 有些只有前端才用的到的Helper，這個有點難拿捏，有些是前台呈現的Helper，但是可能在Schedule Job專案內會需要(產RSS，Sitemap檔案)，那就會面臨參考到Web而不是library，之前有做法是在拆出一個Common專案，專門放前台Web使用的，但也有面臨Common跟Infrastructure在某些Helper上會不知放哪好。
  - A: 最後決定是，Helper類型統一放到 ***Common專案*** 內(放置軟體相關的)，例如EnumHelper類型。把可能共用的都放到這，讓Infrasturcture層放跟infra有關(硬體有關的)的就好
  - 另一個問題製作Extension還是使用Helper呢? 這其實很簡單，Primitive Type本來都有Extension所以有需要擴充的就使用Extension。Helper則使用在自行建立的類上，例如我有一個要組image path的，PagePathHelper。加密編碼的，HashHelper，CryptyHelper，JWTHelper。大眾的屬性Type放Extension，特有類別的放Helper。
- 我的話還會包含attribtue資料夾或是middleware資料夾，再來是OWIN的Option資料夾，JWT資料夾，modelBinding資料夾，Validation資料夾(不包含contract型式的)
- 這裡註記一點，有一個我不常用的，因為這是MVC網站，他可以透過Controller call View/Controller方式
![專案](https://i.imgur.com/s7Neqld.png)


## Site - Application Layer

- 這是目前跨到DDD，最難適應的一層，他主要放置inputModel(有點像是WebAPI我常用的requestModel)，使用商業功能做資料夾區分，因為他會依照你的商業畫面逐步增加
- 參考: IBuyStuff.Domain，IBuyStuff.Domain.Service，IBuyStuff.Infrastructure，IBuyStuff.Persistence
- Services資料夾(ControllerService)，這層就是做分配工作，但他同時也做了Validate，也在這裡建立Return的ViewModel參數，這點是我比較不明白的，他的優點是什麼? 
  - 我猜想MVC專案建置的都採用本文的方式拆分Application Layer，但使用WebAPI的方式就不拆分Application Layer，理由是MVC拆分Application是因為View在MVC專案內，不想把前端跟後端混再一起。而WebAPI則本身就可以是Application Layer，前端採用前三大框架之一的Vue，差異只有在佈署(CD)的時候將前端頁面丟到Server端內，上網找 ***微軟官網*** 也這麼說到，不過他說這是選擇性的喔。![微軟官網](https://i.imgur.com/AK7DOx5.png)
- Utils資料夾內是Extension跟Helper，跟上面的討論一樣，統一放置Common專案
- ViewModels資料夾，就很一般的前台頁面使用model

## Domain Layer

### IBuyStuff.Domain

- 每個Domain為一個資料夾，會依據商業模式逐步增加，每個Domain內包含Aggregate Root，Entity，Value Object，ExceptionEntity(這名稱我自創的，就是當有entity資料錯誤或是找不到時候回傳給他一個錯誤的類)。每個Domain類會有自己的property跟method。建議是會有自己的CreateNew() method，在Product的類沒有CreateNew()我猜是因為他是屬於後台的建立，前台不會建立Product。而這專案只實現前台的樣式，如果包含後台而且共用此上下文，那就可以加CreateNew()到Product類上了，但這樣會違背原本Domain的設計嗎? 畢竟在前台是不需要的，或是拆分成兩種不一樣的Product類可能更乾淨。Value Object不需要CreateNew()
- Shared資料夾內多半放置可共用的Domain，例如Currency。另外Address只有在Customer，我猜是因為Order內是指定Customer，但如果我是購買人，送件地址不同呢? 我就需要多建立一個Receiver了，這樣依樣不需要拉出address。
- 參考: 無
- Repository資料夾，裡面是所有Repository的interface
- IAggregatedRoot類一定要建立，每個Domain的Aggregate Root會繼承他，DDD規則上aggregate才可以存取Repository，才可以跟另一個Aggregate Root溝通，細節可查看[Domain Model這篇](/2019/02/16/DomainModel/)


### IBuyStuff.Domain.Service

- 參考: IBuyStuff.Domain，IBuyStuff.Persistence
- DTO資料夾: 傳送介於ControllerSerive跟Domain.Service層，有一點組合object的感覺，是否可以使用viewModel一路到底，我相信也可以才是，但這樣劃分就不夠乾淨而且目前本文設計會有dll雙向綁定，所以要馬不這麼做，要馬拆分到另一個專案(viewModel到Common專案)。另一個是因為他是搭配entity Framework，如果使用dapper + Store Procedure就可以直接使用viewModel到底了。我目前結論會是不使用ViewModel一路到底，使用別的物件DTO類或是Entity類去接，比照本文的模式。
- Impl資料夾: 實作不在Entity裡面的方法，只要不屬於或是不知道該放到哪個Entity，都可放到Domain Service的Impl資料夾內
- Events資料夾: 建立Event，也就是所謂的事件，需要建立Bus類，有點類似事件處裡器，這很好用，可以用在不斷變化的項目上面，例如: 優惠價格，會員等級。

## Infrastructure Layer

### IBuyStuff.Infrastructure

- 坦白說一開始看到這個Misc資料夾，我還不知道他要做什麼的，Misc全名是啥? 難道是other!!，Misc英文就是雜項。我猜想應該是，所有雜七雜八的都放這。有點Utility的感覺。
在這個範例專案內放了Hashing的類別，會在Application層使用，在Login方法內做Hash密碼用。 後續可以放置Cache，Logger，Queue，Redis (這些都跟硬體機器會有關連)等等。
- 參考: 無
- 另外連線字串ConnectionString跟Config層的部分我也會放在此處，由於日後可以讓其他Persistence共用而選擇不放在Persistence內。目前有遇過Config層作法是使用template方式去刷參數到WebConfig內，優點是可以區分環境，透過Jenkins在CICD時候帶入正確參數，如果你有一個參數要簽入，你必須在所有的環境都簽入，為每個環境的參數帶入正確數值，缺點是在開發階段要去刷這個config很困擾開發。

### IBuyStuff.Persistence

- 這個Persistence就是Repository Pattern該放的地方，
- 參考: IBuyStuff.Domain
- Facade 資料夾類別，主要放置Entity的上下文DbContext。
- Repository資料夾類別，主要放置Repository實作
- Mappings 資料夾，主要放置Entity Framework Code-First需要對應的Enity跟DB Field的欄位設定
- Utils資料夾，主要放置這個層會需要用到的其他項目，像是Seed()，產出範例資料。


# 流程

addToCard()
  - Site.IBuyStuff.Server --> IBuyStuff.Application --> IBuyStuff.Domain

Checkout()
  - Site.IBuyStuff.Server --> IBuyStuff.Application --> IBuyStuff.Domain.Service --> IBuyStuff.Persistence

# 結論

- 在上面的流程過程中，我突然發現，以往我使用的 ***Transaction Script 事務腳本模型*** 跟我後來搭配SPA建構專案，差異在於 ***物件狀態***，使用TS的時候狀態在Session內，本文DDD模式也是狀態在Server side的Session內。 但SPA模式狀態是在Client Side，Server端只需要做儲存跟取值即可(RestFul設計)，其實就TS事務腳本模式即可，或是CQRS即可。範例: addToCard()這個方法就根本不需要進入Server Side，直接在Vue端加入即可，相對我覺得速度更快。



## 來源

1. [**Microsoft.NET企業級應用架構設計**](https://www.books.com.tw/products/CN11327631)
2. [eShopOnContainers 訂購微服務的領域模型結構](https://docs.microsoft.com/zh-tw/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/net-core-microservice-domain-model)
3. [NAA4E](https://archive.codeplex.com/?p=naa4e)
4. [微軟官方](https://docs.microsoft.com/zh-tw/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api)



