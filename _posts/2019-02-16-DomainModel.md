---
layout:     post
title:      企業及應用架構VII
subtitle:   Domain Model
date:       2019-02-16
author:     BY Skyleaf
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - 企業及應用架構
---
# 前言

> 這篇稍微紀錄一下這次讀本書第八篇的感想，這篇講很重要領域模型，我想記錄幾點: ***領域層的內部***，***聚合***。

![图](https://images.unsplash.com/photo-1506345285442-8e9a3a298cdd?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=80)

## Agenda

- 領域層的內部
- 聚合


# 領域層的內部

一般流程，通常我們會依據收到的需求來建立Entity，會以關聯式資料的角度切入，找出每個Entity的關聯跟他需要的服務流程，在這種分析設計模式上通常DB Table跟Entity會是對應的，這是一般我們設計專案模式。再透過分層式架構拆分類的位置，資料存取層，邏輯層，展示層。這種模式在後續的維護上有些困難，當系統邏輯難度不斷提升，維護的成本就相對提高。在我實務上遇到的就是邏輯劃分，因為常常遇到上下文的定義有時候逐漸走偏，可能跟原本的設計不同了，原本設計的人跟後來實作的人也不同了。另一個問題是因應環境快速變化，我認為舊有的設計模式難以符合不斷因應環境需要不斷更新的設計處理，我說明一下會什麼我認為這樣，因為以前設計的時候我只為了符合desktop，沒想過他要符合其他裝置，當時也沒那些裝置，也沒認可的技術可以因應這新的裝置，在於不知道的未來上，某天你可能會需要延展，原本的綁定上下文的限制就可能綁住你的延展。另一個是不斷堆疊的邏輯，在沒有劃分清楚的情況下，當堆疊到一定程度很容易進入大泥球。
DDD設計原則，首先是統一語言，在建構綁定上下文，最後以業務領域對象模型為中心去做分層式架構。拆分的顆粒度會以你團隊目前的狀況來細分，再慢慢因應需求的變更，不斷拆分你的顆粒度讓他越來越符合你的需球。這裡想說的是你很難一次到位，當然你越是熟悉你的Domain就越是能接近你的需求，這裡講的拆分也包含在耦合上，在關聯上，不只是Domain的拆分，有點分散式設計架構。但這樣才能因應快速變化的科技，每個新增每個修改每個異動都能不影響現況，做到持續整合持續部屬。這邊我想說到另一個問題就是DB，多半在軟體部份DDD，微服物等等已經是一個可預知的目標，達到我相信是時間的問題，在硬體方面也有S2D的技術虛擬化解決硬碟的限制。但不要忘記DB，DB的顆粒度拆分，DB也需要因應未來的不斷變更，我相信DB不拆分很難達到分散式設計架構，瓶頸會在這。當然也是有解的，這得再微服務篇再說道。但在做到那之前，你需懂得如何透過DDD原則設計一個上下文，不同上下文可以搭配不同符合需求的設計框架。回到Domain Model，這裡要講幾個非常重要的Key，不搞懂這幾個的定位你會很難做出領域模型。

Domain Model 領域模型由實體跟值對象構成，它的結構還包含: 領域模型，模塊，領域服務。

## 1.Module 模塊

模塊式用來分組用的，有點像是namespace命名空間，優點是可以更加的精處區分領域之間，以下列出本文透過模塊可能的關係:
- 一個綁定上下文有一個領域模型
- 一個綁定上下文可以有多個模塊
- 一個領域模塊可以關聯多個模塊

## 2.Value Object值對象

模塊裡面包含實體跟值對象，值對象也是一個類，它的特性是immutable不可變的，只能透過新增一個實例instance去改變，也就是一旦創建就不會改變。

## 3.Entity 實體

實體的特性是它一定有ID，一個身分標示。實體跟值對象最大的差異是實體是由數據跟行為構成，而值對象是組合再一起的數據。實體會去觸發Repository做持久化，不會自己實作持久化，每個Entity都有自己該負責的邏輯方法，跨越多個Entity的商業邏輯方法則由Domain Service代勞。

## 4.實體的持久化

實體要做持久化就需要透過Repository倉儲，而Repository的實做會放在基礎建設層，Repository的Interface會放在領域層。

# 聚合

有些Entity基本上就是一組的，他獨立存在是沒意義的，例如: OrderItem，他通常是跟著Order的。聚合也是一種分組，跟Module一樣，但Module分的是層級比較大，而聚合是分組模型，也隔離實體，有些實體是不因該被直接存取的，他因該透過Aggregate Root去存取，例如Order，Aggregate Root也是一個Entity。而Aggregate通常就是對應到Repository的Entity，這裡我以前常搞混，Repository的Entity對應到DB的Table，而Domain存取資料因為透過Aggregate組合，會只能透過Aggregate Root去存取Repository，所以Aggregate Root也因此會對應到最終的DB Table。我覺得先有這樣的概念跟劃分，後續在實作上才能變化Aggregate Root跟DB Table的對應。我在想他是可以不ㄧ定百分百的對應的。也有些Entity並不是Aggregate Root卻可以被其他的Aggregate Root引用，例如: Product，他會能對應到OrderItem。

聚合是一種設計方式，他用來表示業務模型裡的***不變條件***，也就是業務的規則，他也是一種一致性的保證。一致性可以區分兩種: 事務一致性跟最終一致性。

事務一致性: 保證每次發生在這個業務事務的結果都一樣
最終一致性: 他的保證是最終會一樣，但不是每次業務事務結束都一樣

優點: 邏輯比較乾淨，因為每個模型劃分清楚，在實現上比較可以專注。每個實體都有自己的行為，透過聚合，限定其他私有建造，避免外部隨意引用，可保持一致性。在整體的耦合上比較鬆散。這邊的意思是一個Entity可以引用另一個Aggregate Root或是同一個聚合內的Entity，但是一個Aggregate Root只能引用另一個Aggregate Root不能直接引用別的聚合內的子Entity。也因為這樣的原則劃分，才可以更乾淨的觸裡每個模型。

# Aggregate Root 聚合根

在上面提到蠻多了，Aggregate Root可以被直接引用。想要訪問這個聚合底下的子Entity就必須透過這個聚合的Aggregate Root。Aggregate Root也負責要去聯繫持久化Repository。

## 來源

1. [**Microsoft.NET企業級應用架構設計**](https://www.books.com.tw/products/CN11327631)
2. [eShopOnContainers 訂購微服務的領域模型結構](https://docs.microsoft.com/zh-tw/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/net-core-microservice-domain-model)



