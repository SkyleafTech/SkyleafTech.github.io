---
layout:     post
title:      企業及應用架構XII
subtitle:   CQRS
date:       2019-04-09
author:     BY Skyleaf
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - 企業及應用架構
    - Event Sourcing
    - Saga Pattern
---
# 前言

> 這篇稍微紀錄一下這次讀本書第十二跟十三篇的感想，這篇想記錄Event Sourcing，事件來源或是事件溯源

![图](https://images.unsplash.com/photo-1532751788314-cf521c13ad75?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=607&q=80)

## Agenda

- Event Sourcing
- Saga Pattern

# 事件溯源 Event Sourcing


在看這篇之前，我在以前做過一個物流系統，當時我就在想，每筆進出單是屬於單據，庫存的數量我記錄在一個名為庫存的表內，使用的是關聯式資料庫，當時我遇到一個問題是，客戶需要可以連覽某個時間點的進出明細，以及當時的庫存數量，我有一個庫存表她會記錄每個商品的目前庫存數量，而進出單可以當作是進出明細的紀錄，但當客戶想要查某個時間區間的時候我無法往前或是往後推算當時的庫存數量，因為我並沒有紀錄某個時間點的當下庫存點，當時想到那就把該商品的庫存重頭算到選取的時間就好了啊，但當資料很龐大時候效能會很差。變相只能要求選取的時間不能過遠。有點像是銀行那樣只給你選6個月前，最多。

另一個做法是每次在進出單內就作加總，那我就可以知道當時進出的數量跟當前總數，如果要重算我都可以抓出該區間，取出當時的當前總數欄位即可，這方法可以解決選取區間的問題，但當我需要更正數量的時候就麻煩了，我必須把往後的資料全部做異動，也想只透過加的方式，加在最後補上要異動的數量差額，但就會出現中間區間的髒資料留存，雖然解決的最終一致性，但也保存了髒資料。

始終覺得美中不足，直到我看到了這篇Event Sourcing，很久之前在聽到某大大在夏威夷介紹這個的時候，他也使用了銀行的範例，說道我們都會記錄發生了什麼事件，銀行的事件就是交易紀錄，他不會記錄最終Account Balance，他是透過事件，重新呈現目前的Balance。這方法叫 ***Event Replay 事件回放***。而他也說到當資料量大的時候，你可以使用 ***snapshot快照***，對就是這個快照，當時我還沒連貫起來我的物流系統，現在想想，我就是需要快照來解決我的**選取區間數量**的問題。

事件溯源 Event Sourcing: 你紀錄的是事件，持久化消息，追蹤應用程序的狀態，而不是覆蓋結果，他更有效的紀錄了發生什麼事 (traceable)，如果你的系統需要知道過程發生了什麼狀態，你就需要Event Sourcing。另外先說到Event的好處，他可以讓你在任何時候添加事件到任何的domain上，而不影響原本的流程(Scalable)。事件可以去除模型邊界的限制，這是說在你的聚合內，裡面是你的綁定上下文，以購物車為例order是聚合根，裡面有product跟orderDetail，如果你有新需求，這個聚合勢必要做異動，但搭配event就可解決原本的上下文限制，假如我的order內商品包含可安裝的服務Service，這服務可能會是配送方需要知道的，除了直接加到這個聚合，異動這個聚合之外，如果使用Event，讓application層在分配的時候觸發服務即可。 還有一個優點是假設場景，也是我最上面遇到的需求，我需要呈現某個時間區間的資料流狀態 (Event Replay)。

### 三個特性: 
1. 包含事件對象，重建此事件的狀態跟訊息
2. 必須可以返回，跟保留關聯的key
3. 只能add，不能update或是delete


### 在執行領域事件上: 

- 如果在聚合根內直接帶入要發佈的事件，但這樣的缺點是你的聚合根就必須帶入要發布的事件，是種汙染聚合根了, 另一個方式式NServiceBus的創辦人Udi提供的使用靜態方法發布領域事件，但是這樣的缺點就是不易做測試。

- 最後採用LosTechies網站的建議，在聚合根內 ***臨時保存領域事件***，注意喔這種方式的聚合根所保存的事件必須是臨時的，該聚合根操作完就清空，這跟企業架構本書的ERP專案的方式有點像。另一種跟 ***臨時保存領域事件*** 相似的是“在聚合根方法中直接返回领域事件”，也是好測試， 但作者說缺點要注意的是"不適合創建聚合根的場景，因為要返回聚合根本身也要返回事件"，這句話我不懂!!我猜想是複雜度提高了，讓一個方法多做了一個以上的事物。

### 業務操作跟事件發布的原子性 

- 聚合更新和事件发布之间的原子性，內容表示可以把兩件事都存在同一個關聯式的資料庫，透過更新聚合資料，發布訊息狀態改成已完成，來達成原子性，但缺點在於當某一方fail的時候，如何達到，可能需要有後端系統定期查詢重送。
- 事件也最好是"幕等"的，消耗方重新執行就不會有問題。
- 用MongoDB的Document保存聚合根便是种很自然的方式，NoSQL比關聯式好
- 另一個方式: 聚合根序列化成JSON格式的数据进行保存


# Saga Pattern 介紹

根據Bernd Rucker介紹Saga就是"The Saga has the responsibility to either get the overall business transaction completed or to leave the system in a known termination state"。我後來查到他其實是一個分散式交易用來處裡一個長時間的商業流程，當中可能會參與多個聚合。

兩種方式實作: 
1. Orchestration
    - 這此文章Bernd表示他建議使用Process Manager (Orchestration)，也就是所謂的有中心協調者broker，每個服務透過一個協調中心broker。但缺點是單點故障問題，我猜想是因為都透過broker。另外Bernd推薦搭配Camunda工具(Open Source BPM platform)，可以查看整個Business流程，可以視圖化。
    - 另一篇[Couchbase公司的文章](https://blog.couchbase.com/saga-pattern-implement-business-transactions-using-microservices-part/)解釋Orchestration解決了在Choreography會有的雙向depencies問題，中心化操控分散式交易，更方便實作跟測試，方便擴展，更易rollback，更方便操控交易順序，當然還有解偶。
    - Command/Orchestration: "when a coordinator service is responsible for centralizing the saga’s decision making and sequencing business logic"
    - ![Choreography](https://blog.couchbase.com/wp-content/uploads/2018/01/Screen-Shot-2018-01-11-at-7.40.54-PM-768x470.png)

2. Choreography 
    - Events/Choreography: "When there is no central coordination, each service produces and listen to other service’s events and decides if an action should be taken or not."
    - 無中心協調者，每個服務沒有協調點，都是靠服務自己相互直接協調
    - 優點: 解偶
    - 缺點: 要小心雙向dependency，太多的steps有時候很難track，不好做測試
    - ![Choreography](https://blog.couchbase.com/wp-content/uploads/2018/01/Screen-Shot-2018-01-09-at-6.13.39-PM-768x817.png)


# 結論

在這個Event Sourcing當中，以往我認為CQRS當中的讀，是可以簡化到只讀取快取資料，但是如果要查看資料的變化狀態，就必須使用事件溯源，當然不是每個系統都需要事件溯源。如果要達到查看資料的變化狀態，就需要使用事件回放，而考量效能問題，就需要搭配snapshot快照功能。

之所以要在這篇加上Saga是因為想補上前面的CQRS篇的實現命令線。先前說道的領域事件跟集成事件，都是在command發生後需要告知的事件動作，但執行上有所不同，領域事件是推送消息到總線，集成事件是讓綁定的上下文相互通信。這裡就聯想到Orchestration跟Choreography。




## 來源

1. [**Microsoft.NET企業級應用架構設計**](https://www.books.com.tw/products/CN11327631)
2. [eShopOnContainers 訂購微服務的領域模型結構](https://docs.microsoft.com/zh-tw/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/net-core-microservice-domain-model)
3. [NAA4E](https://archive.codeplex.com/?p=naa4e)
4. [微軟官方](https://docs.microsoft.com/zh-tw/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api)
5. [CQRS](https://www.codeproject.com/Articles/555855/Introduction-to-CQRS)
6. [浅谈命令查询职责分离(CQRS)模式](http://www.cnblogs.com/yangecnu/p/Introduction-CQRS.html)
7. [Berndruecker](https://blog.bernd-ruecker.com/saga-how-to-implement-complex-business-transactions-without-two-phase-commit-e00aa41a1b1b)
8. [Couchbase公司的文章](https://blog.couchbase.com/saga-pattern-implement-business-transactions-using-microservices-part/)
9. [在微服务中使用领域事件](http://www.cnblogs.com/davenkin/p/microservices-and-domain-events.html)



