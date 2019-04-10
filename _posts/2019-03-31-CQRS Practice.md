---
layout:     post
title:      企業及應用架構XI
subtitle:   CQRS
date:       2019-03-31
author:     BY Skyleaf
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - 企業及應用架構
---
# 前言

> 這篇稍微紀錄一下這次讀本書第十一篇的感想，這篇想記錄EventBus，讀寫分離後，讀變得相對簡單了，但寫會推薦使用EventBus/CommandBus的方式，讀跟寫也是使用不同的資料源

![图](https://images.unsplash.com/photo-1545424436-ebfb08353407?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=80)

## Agenda

- EventBus


# 事件源 EventBus

![CQRS](https://www.codeproject.com/KB/architecture/555855/CQRS.jpg)

在這裡還是要補上這張圖，架構大概也就是這樣，為啥要使用EventBus，在第八章其實有說道在很常變化的邏輯上，使用Domain Event可以降低程式碼的複雜度，但你得先懂得如何使用。這邊想歸類幾個必要的東西:

1. Base
   1. EventBus.cs
2. EventHandler
   1. SendEmailEventHandler.cs
   2. SendMessageEventHandler.cs
3. EventData
   1. ImportItemsEventData.cs
4. Interface
   1. IEventBus.cs
   2. IEventData.cs
   3. IEventHandler.cs
   4. IEventStore.cs
5. EventStore
   1. InMemoryEventStore.cs

![EventBus](ttps://i.imgur.com/mhneeYO.png)

## EventStore跟EventBus
最主要的就是EventBus.cs，裡面會有基本的Subscribe，UnSubscribe，Publish這三個方法，可以更多但一定會有這三個，名稱當然可以不同，不過就是要Subscribe註冊，註冊你的事件跟處理這事件的handler，還有就是觸發Publish。你可以註冊到In Memory的原件例如Dictionary<Type, List<Type>>()，非同步可以使用ConcurrentDictionary<Type, List<Type>>，如果不喜歡In Memory，你可以選擇RabbitMQ，NATS等等Message Queue的套件。這裡你一定想問我哪時候該用哪個阿，這麼多，一開始我建議你先別用第三方的Message Queue，不然你還得花時間去研究。不過等你搞懂候你就會需要。不外乎解決延展性跟耦合。一開始你也不需要那些IEventBus跟IEventStore也可以達到EventBus的製作，我之所以建立是因為後續會需要串接第三方套件時候抽換比較好抽。這裡要說明一下EventStore也是可以最後套用第三方套件，有一套就叫做[Event Store](https://eventstore.org/)，他是.Net寫的，有興趣的朋友可以試試。由於我分開了EventBus跟EventStore所以裡面都會有Subscribe，UnSubscribe這兩個方法。

## EventData跟EventHandler
再來你會需要製作EventData，有的人使用xxxEvent，我的後贅詞我使用EventData。裡面大部分會有的就是Id，透過這個來做處理。我使用ImportItemsEventData類來作當我進了一批貨，在我的Service層會去觸發Import該Items的數量，完成後我會使用EventBus.Publish()方法去觸發有註冊到ImportItemsEventData的Handler。這裡我建立了兩個Handler: SendEmailEventHandler.cs跟SendMessageEventHandler.cs。 一個是執行Email寄送，一個是傳送訊息。你一定要IEventHandler讓你的handler繼承，裡面會有一個方法，這方法內就是該hanlder要執行的邏輯。另一個是IEventData，你可能想問為什麼需要這兩個Interface，因為會需要對應，而透過interface方式你才可以抽象化，不然你綁太緊你很難抽換。

## IOC初始化你的類
最後要講到怎麼對應，以前我們會用XML，把EventData寫出來，跟他會用到的Handler寫出來，在程式碼一開始時候initailize()，建立對應。我的範例就是ImportItemEventData會對應到SendMessageEventHandler跟SendMailEventHandler兩個。所以當我使用EventBus.Publish()觸發ImportItemEventData的時候，他會去EventBus內的EventStore的In Memory的Dictionary<Type, List<Type>>()原件抓出我一開始建立的對應，然後loop符合的handler作執行。另一個方式是使用IOC，在一開始的時候注入你要對應的類別，後面再使用的時候可以透過IOC方式取出該類做觸發handler。這裡有兩點要講，第一個是使用IOC的好處，你不用一直new物件，可以重複使用一個就好，反正hanlder處理的方式一樣，差別是你帶入的參數。再來是使用Dictionary<T, List<T>>方式，他是為了讓你區隔哪個EventData註冊了哪些Handlers。第三個是找出handler類別，基本上不使用Interface，你會把你的程式碼寫得很亂，完全無抽象，透過interface，可以抓取有繼承他的子類別，在擴充方面才可以依據interface方式延伸。



# 結論

- EventBus的優點在我認為上就是縮減複雜度，原本一個會員註冊的邏輯跟註冊完後的邏輯都是寫在一起的，但透過EventBus的方式可以抽離這些邏輯，後續修改上，可以依照顆粒度來修改，異動到同一個類同一個方法總是有風險的，如果可以拆分更細的方法到其他的類上，在修改上也比較符合開放封閉原則。本文也有說道，切忌不要濫用EventBus的方式，過度使用反而未必好維護，所有做架構最終目的都是希望後續好維護，而評估的考量當然還是回歸到團隊上。



## 來源

1. [**Microsoft.NET企業級應用架構設計**](https://www.books.com.tw/products/CN11327631)
2. [eShopOnContainers 訂購微服務的領域模型結構](https://docs.microsoft.com/zh-tw/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/net-core-microservice-domain-model)
3. [NAA4E](https://archive.codeplex.com/?p=naa4e)
4. [微軟官方](https://docs.microsoft.com/zh-tw/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api)
5. [CQRS](https://www.codeproject.com/Articles/555855/Introduction-to-CQRS)
6. [浅谈命令查询职责分离(CQRS)模式](http://www.cnblogs.com/yangecnu/p/Introduction-CQRS.html)



