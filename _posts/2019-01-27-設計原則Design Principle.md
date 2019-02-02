---
layout:     post
title:      企業及應用架構III
subtitle:   設計原則Design Principle
date:       2019-01-27
author:     BY Skyleaf
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 企業及應用架構
    - Design Principle
---
# 前言

> 這篇稍微紀錄一下這次讀本書第三篇的感想，重點應會想講SOLID但是我想記錄別的， ***Composition vs. inheritance*** 另一個是 ***Defensive Program***。

![图](https://images.unsplash.com/photo-1490261704701-28a81653095c?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=80)

## Agenda

 - Composition vs. inheritance (組合與繼承)
-  Defensive Program (防禦性編程)

## 前提

- 組合與繼承: 主要講繼承會遇到的一個設計上缺點，透過組合方式來解。
- 防禦性編成: (Defensive Program): 這就是在你的每個方法入口加入輸入值得驗證，確保任何可預期的狀況提早驗證，避免後續例外發生。

# Composition vs. inheritance 組合與繼承 :

### 1. Abstract繼承的問題:

   之前有被問到abstract 跟 interface的不同，當時我沒好好解釋，我只說了層級跟層級的溝通我會使用Interface來做接口(低耦合)，重複的程式碼，例如CRUD我會抽出來放到abstract內(高內聚)。但就在這時候我突然想到abstract其實有個缺點，這缺點就是他的異動引響很大，有繼承他的子類別都會影響到，雖然實作可能用不到或是根本不會觸發，但設計上是有漏洞的。本章節剛好解釋這個問題，內容說到須確保父類跟子類可以交替使用，而這正是里氏本原則(LSP)來的目的。當我使用abstract類時候，是因為我有程式碼會重複，所以我想抽出來到一個父類上，讓其他的子類都可以使用，當中如果有重複的property也可以抽出來。但是限制就是繼承的子類一定會繼承到這些property跟method。那如果用不到呢? 

### 2. 組合:

   這裡必須提到GoF的大神們說到程式碼重用的兩個方式:　白盒跟黑盒。
   - 白盒: 是指基於 ***類的繼承***
   - 黑盒: 是指 ***對象組合***

   看到這篇讓我想到黑盒難道是那個black-box reuse? 
   
   這邊我上面講的CRUD抽離到父類的方式，就是白盒，所謂的類的繼承。因為愛用Repository Pattern的我常會使用到這種設計原則。但是遇到上面說到的問題就不能再沿用這種白盒方式設計建構，需要使用黑盒的方式來解，也是所謂的 ***對象組合***。你需要建立一個新的類，透過這個類(包裝類)去建立想使用的對象類別(被包裝類)。而且多半會透過private方式引用建立。本文稱這種組合式的類別為 ***包裝類***，該類不能訪問此 ***被包裝類*** 的內部，也不能以任何方式改變此 ***被包裝類*** 的行為。他只能使用這個 ***被包裝類*** 而不能改變此 ***被包裝類***。外層透過這個包裝類來處理內部的對象，執行某種特定行為。內部的對象也就是被包裝的類，可執行哪些 ***被包裝類*** 的方法則由開發者自行決定。 Composition的另一個Inheritance沒有的優點就是它是一種Defensive Programming。下面解釋這是啥。不過整體來說這個方式可以解決一些OOP的設計缺陷。不過如果你要 ***被包裝類*** 跟 ***包裝類*** 都使用到其實可以使用Interface方法讓這兩類都繼承這接口，再透過接口使用。不過這取決於你的如何定義類的關聯。 

   這邊想說的不是inheritance不好或是他一定得用composition來解救什麼的，在某種設計狀態下，你會需要使用composition，而某種狀況下的composition也會有需要inheritance來解的狀況。


### 黑盒範例: 

   在我的成立訂單上，我覺得就是一個組合類，Entity對應到DB資料結構，但DB裡面是不會有ShoppingCart這個table的，也不需要。但在頁面跟application層內會需要ShoppingCart這個組合類，他會包含Product(entity)，Coupon(entity)，Member(entity)。多半設計上都使用Order搭配OrderItem的設計方式。會有點跟歷史訂單搞混，感覺使用Order跟OrderItem也可以達到流程需求，只是語意上怪怪，我們都說購物車，但是商業邏輯domain的物件上卻看不到shoppingCart的類。如果ShoppingCart物件內包含了Products，包含了Member資料，包含了Coupon在Domain區塊上。從Controller開始組資料。BL層做商業邏輯處理，到了DAL層再逐步處裡每個Entity的寫入，例如Order，OrderItem，Product Inventory，Coupon。
   但有一個很大的問題就是，以往使用的3層式架構CRUD設計要套用DDD是有困難的，完全不同的概念，這也是一直搞混我的地方。

   ```
   後續補上
   ```
 

# Defensive Program (防禦性編成): 

### The If-Then-Throw Pattern

   這是很常見的方式，你會在方法的一開始使用如果傳進來的input沒包含某某值throw Exception來防堵後續錯誤。

## Software Contracts

   他就是一種對方法的限制條件，只有在某種符合狀況才可以進入執行該方法。本書說到.NET在4.0後可以透過套件Code Contracts使用，我測試了一下.NET Framework 4.6.1在System.Diagnostics.Contracts內有這個方法。而這個方式比較進階，我也沒用過，在.NET使用Code Contracts實作，他區分程三步驟: 前置條件，後置條件，不變條件。

### 前置條件

   這個跟The If-Then-Throw Pattern基本上差不多。都是進入點判別值是否符合方法需求。之前我WebAPI都是透過Model Binding方式驗證，甚至有的共用同一個request物件，在不同方法搭配不同model binding規則方式(其實建立另一組新的request物件即可，但還是被我試出來了XD)。使用這個跟使用model binding差異不知道是什麼? 有空再查查。
   
   ```
   Contract.Requires<ArgumentException>(string.IsNullOrEmpty(value));
   ```

   
### 後置條件

   前置條件是管進來的值，而這個是管輸出的值或是狀態。要馬彈出例外，要馬驗證成功。這裡說道如果驗證的值需要回傳，可能需要暫存在其他處，因為回傳只傳bool值。
   
   ```
   Contract.Ensures(string.IsNullOrEmpty(value));
   ```


### 不變條件

   不變條件比較嚴苛一點，他會驗證在執行期間該類的property是否改變，只要改變及觸法不變條件，但這樣很困擾的，雖然你可以確保他都是符合你的期待。 優點是不必在其他地方判別該property是否符合，只需要在該類別加入attribute跟規則即可。困擾點是，稍微不符合輯觸發，不管影不影響流程，他綁的是物件property。不過看需求需要來決定才是。

   ```
   Contract.Invariant(string.IsNullOrEmpty(TotalPrice));
   ```          

## 來源

1. [**Microsoft.NET企業級應用架構設計**](https://www.books.com.tw/products/CN11327631)
2. [**Microsoft .NET: Architecting Applications for the Enterprise**](https://ptgmedia.pearsoncmg.com/images/9780735685352/samplepages/9780735685352.pdf)


