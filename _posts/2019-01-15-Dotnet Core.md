---
layout:     post
title:      Dotnet Core
subtitle:   Dotnet Core 初探
date:       2019-01-15
author:     BY Skyleaf
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Dotnet Core
    - LineBot
    - .NET Core
---
# 前言

> 這篇稍微紀錄一下這兩次參加的研討

![图](https://images.unsplash.com/photo-1520509414578-d9cbf09933a1?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=649&q=80)

## Agenda

 - Dotnet Core MVC
 - LineBot

### 前提

- 這兩次是由聖殿祭司(奚江華老師)跟董大維老師兩位主辦，第一次的研討董老師生病，聖殿祭司一間扛起整個下午，個人覺得超屌，果然是有實力有準備。內容帶到很多.Net Framework MVC 跟目前.Net Core的差異。需要注意什麼跟基本的Demo。第二天董老師講的是LineBot的一些新功能。但不得不說奚老師還請喝飲料，真是佛心。他覺得週末肯出來聽聽研討，輕鬆一下。

-時間: 2018-12-02 跟 2019-01-14兩場

# 聖殿祭司.Net Core內容
 
### 聖殿祭司認為最需要注意的主要四項:
1. WebHost
2. Configuration
3. DI
4. Middleware

#####  名稱很容易搞混:
- .Net Core 對應 .Net Framework
- ASP.NET Core 對應 ASP.NET Framework
- ASP.NET Core MVC 對應 ASP.NET MVC

##### 安裝:
- 基本上有三種的樣子，我使用VS 2017需安裝 .Net Core SDK 就對了，他全包
- 透過CLI 命令做建置，或是使用VS2017 (直接包在裡面了)

##### 專案架構:
- Content Root:整個專案的最上層就是Content Root
- Web Root: 就是放靜態檔案的，因為以前我們Deploy只能放到IIS目錄，但是現在要在任何folder位置都可以跑。所以他被整理出來跟著專案。
- .Net Standard Library 統一
##### Web專案選項:
- MVC: 這個包含的項目最多
- WebAPI: 這個沒有View這層
- Razor Page: 這個沒有Controller層，而且他有點像是以前的codebehind結構，他也沒有Router機制，他是跟以前一樣用檔案對應方式

Fundamental: 講師認為這次改最大的就是底層核心架構，以往以前都包好了，現在全部開放可以抽換修改設定。
- 例如: Logger。在ASP.NET MVC是沒有的，都是用第三方套件Log4Net等等的，現在.Net Core已經有內建了

有好多個，這邊先列有紀錄的
- Environment: 可區分Dev，Stage，Prod
- 透過launchSetting.json檔案設定Host專用的參數，例如port位置
- 透過AppSetting.json檔案設定App會使用的變數，例如ConnectionString

##### WebHost:
- 有兩種Type: 
  - WebHost: 建議使用，跟Web App有關的都在這
  - GenericHost: 不是Web App的，講師說未來可能會使用
- WebHost: 主要在做Configuration跟Middleware
- 先在Program.cs內做Configuration
- 在來Startup.cs做Middleware相關
- 如果要查看Configuration吃到那些參數，把原本createWebHostBuilder()拆分成兩個步驟透過Debug去看
- 你也可以override原本Kestrel內設定的參數透過IWebHostBuilder。
- 修改內部參數可包含bind到IIS，Logging等等

- Lifecycle: 
  - donet run 
    - 注意這步驟會block Thread。也就是他run起來後，你沒辦法再run
  - launchSetting.json
  - 待

- IApplicationLifetime可以區分另外3個動作，分辨可在這三個順序動作做想要做的事。EX: xxxed，xxxing，還有一個沒記到

##### Configuration:
  - 在使用CLI下command的參數，他就是帶到program的main的args內。所以command帶的參數可以複寫你原本設定的參數。EX: port的位置
  - 兩種載入configuration方式: Webhost跟configurationBuilder
  - 載入後會被存在記憶體內，會變成key value格式，注意key可以重複，可是最後一筆會覆蓋掉前面的key值，可且還可以設定ReloadUnchange=true我猜可以runtime修改它

##### Unit Test
- 就是要開始寫Interface
- 寫Interface也才可以做Unit Test。遇到注入的可以才可以透過interface使用- Mock，而Stubing可以用在Repository Pattern上
- Unit Testing有幾件事做不到:
  - Performance效能證明
  - 沒有深層Test證明，只有被測到的參數範圍可確定
  - 沒有壓測證明

##### Dependency:
- VIEW也可以使用Inject 物件，就可以使用該Interface定義好的Properties，對我一直很少用Interface使用properties
- DI的Instance生命建構
  - Tranient: 整個Request的Lifetime有用到3次該物件就會new 3次
  - AddScope: per Request
  - AddSingleton: 系統起來時
- .Net Core內建DI有自動Dispose()方法

#### Middleware:
- 這部分我完全沒去記，因為對很常寫WebAPI的我來說很熟悉。所以我猜轉移到 .Net Core的middleware應該是差不多

#### 最後

- 講師建議: .Net Core 2.0 不考慮，直接使用 .Net 2.1或是2.2。但2.2支援的太新，怕你要Migrate的專案內容版本會不相容

MVC 5 migrate .Net Core 2.1 差異
- 新的 .Net Core他們把牠放到/wwwroot/內
- .Net Framework頁面的Html Helper被移除了，他的缺點就是要到Runtime Render才會知道。.Net Core採用Tag Helper (不知道是否Compile就可知道錯誤)。
- Bundle跟Minify移除了，原本坐在Server Side，改用CDN載入Script

DI Demo:
- View也可以做DI，只是講師建議思考思考看看，透過@inject Ixxx xxx 做注入
Action 也可以DI需要加上[FromServices]即可
- 只要有使用DI，優點是它會自動幫你處理好Dispose()


## LineBot

範例: [小P記帳APP](https://github.com/isdaviddong/Linebot-Demo-AccountBook)，[人臉辨識](https://github.com/isdaviddong/Linebot-Demo-FaceRecognition)。

### LineBot跟APP比較: 

- 老師認為APP建置成本過高，而且須建立IOS跟Andriod兩個版本，上架審核時間。
- 使用LineBot很適合在台灣使用，大部分台灣人都有Line帳號。可使用在請假，點名，報到等等系統上
- 台灣目前1800-1900萬人使用Line
- 人臉辨識是搭配，Azure Vision，Translate，QnA或是Luis (machine learning)。LineBot可以透過call Luis的api取得機器學到的結果

#### Line新功能: 
- QuickReplay: 他是一個功能選單，可以避免nlp (我猜是使用打錯之類的，透過這個可以統一)，可輸入指定格式，例如日期，地點。
    - 圖片icon建議使用npg，連結位置要使用https
    - 最多可以加到13個功能
    - ![QuickReplay](https://i.imgur.com/y2SHp8R.png)


- LIFFWeb: 他是一個可以崁入web的功能，就可以提供form的格式給使用者
    - 三種格式
    - ![Flex Message](https://i.imgur.com/VInBsGP.png)
    - 表單模式
    - RWD網站的方式崁入到這個模式裡面，只需要在Line Developer內設定好網站連結即可
    - 改模式是Line的一種Browser，所以有些限制在Browser。沒有像Chrome這樣強大
    - 優點是: 填寫表單這類功能可以不用離該Line App
    - 建立LIFF必須透過channel在Line Develop Page內
    - 最多可建立20個LIFF，如果真的滿了，董老師建議可以透過QueryString方式來區分更多功能。砍進去其實很像是WebView的感覺
- Flex Message: 一種訊息格式 with Template
    - Simulator 產出的JSON檔案，必須要再包在[] ，array格式裡面，才能送
    - 董老師的SDK內會幫你把貼到c#轉成單引號的再轉回雙引號，適合用在簽到跟需QR Code顯示功能上。
    - ![Flex Message](https://i.imgur.com/4fw7kh0.png)

#### Demo:
- 使用.Net Core + Web Form + Web API + Nuget的LineBotSDK
- 開發階段，董老師建議使用ngrok來轉localhost到你的網址(超好用)



## 閒言閒語
1. 老師的相關課程[**ASP.NET Core MVC2.2 課程**](https://www.codemagic.com.tw/)


#### 來源

1. [**ASP.NET Core 2.1設計新思維與新發展**](https://www.slideshare.net/dotnetcool/aspnet-core-21-125035257?fbclid=IwAR0ELuJgACmnnUOCegSMeTGRqUVdrLJP6NEuwHU8hmr5Rz7SCWLNmocP364)
2. [**ASP.NET Core 從陌生到熟悉 Part II**](https://www.slideshare.net/dotnetcool/aspnet-core-mvc-22)
3. [**使用ASP.NET Core 開發 LINE Bot**](hhttps://isrock.kktix.cc/events/01122019)

```

```	
		

