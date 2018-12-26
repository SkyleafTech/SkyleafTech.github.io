---
layout:     post
title:      Windows Container
subtitle:   Windows Container
date:       2018-11-25
author:     BY Skyleaf
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Windows Container
    - Docker
---
# 前言

>初探容器化技術，這篇想記錄的是使用Windows Container，對，不是Linux，我知道Linux得比較成熟，資源也夠多，文章也夠多，但也因為這樣才想說紀錄一下，不過主要原因是俺是靠微軟吃飯的，能多了解就多了解。


![圖](https://images.unsplash.com/photo-1494412651409-8963ce7935a7?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1050&q=80)
# 操作方法介紹

- Windows 10 + Windows Container
- Windows Server 2016後 + Windows COntainer

#### 操作須知

1. Windows 10 (Version 2.0.0.0-win81 (29211))
2. Windows Contain CE (Version 2.0.0.0-win81 (29211))
3. Cmder (Powershell)


#### 操作想法

這篇主要紀錄如何將你的.NET的Web做Build然後Deploy到IIS上，網路也有大大教學怎樣做，只是他分開了build跟deploy，而我想要的是，把build跟deploy全部放到一個dockerfile裡面處理。對就是比較懶一點XD。因為目前我還不敢把公司的Production CICD流程改用docker，主要是因為使用Windows Contianer，沒人有信心維護，也沒時間。但是在dev階段我覺得還是可以試試看的。其馬對往後使用Windows Container的日子上有點概念。

主要流程是我本機MVC專案裡的程式碼改好後。透過Dockerfile去觸發建置，建置好後的dll再丟到另一個container內做deploy到IIS的動作，整體的流程只需要一個dockerfile。有興趣的朋友可以參考參考，不得不說目前Windows Container的雷真多。

#### 操作工具

1. Docker
2. Dockerfile
3. PowerShell
4. CMD
5. dontframework
6. msbuild
7. IIS

# 流程 最終完成版本:
1. 使用dotnet-framework:4.7.2-sdk-windowsservercore-1803版本
2. 透過multi stage技巧，所以只需要一個Dockerfile完成build跟deploy
3. 目的是要build WebApi專案，所以沒丟test專案
    ```
    FROM microsoft/dotnet-framework:4.7.2-sdk-windowsservercore-1803 AS build
    WORKDIR /app
    ```
    a. 針對csproj檔案來依序建置

    ```
    COPY *.sln .
    COPY WebAPI/*.csproj ./WebAPI/
    COPY WebAPI/*.config ./WebAPI/
    COPY Domain.DataClass/*.csproj ./Domain.DataClass/
    COPY Common.Utility/*.csproj ./Common.Utility//
    COPY Common.Utility/*.config ./Common.Utility/
    RUN nuget restore
    ```	
    b. 先使用nuget restore
    
    c. build的過程需要再丟其他檔案(頁面的)
    ```
    COPY WebAPI/. ./WebAPI/
    COPY Domain.DataClass/. ./Domain.DataClass/
    COPY Common.Utility/. ./Common.Utility/
    ```	
    d. 在逐一建置專案
    ```
    RUN msbuild /p:Configuration=Release ./Domain.DataClass
    RUN msbuild /p:Configuration=Release ./Common.Utility
    RUN msbuild /p:Configuration=Release ./WebAPI/
    ```	
    e. 接著準備Deploy環境
    ```
    FROM microsoft/aspnet:4.7.2-windowsservercore-1803 AS runtime
    ```	
    f. 接著要把IIS的32bit給打開，因為Oracle是使用32bit的，有兩種方式cmd或是powershell
    ```
     RUN C:\Windows\System32\inetsrv\appcmd set apppool /apppool.name:DefaultAppPool /enable32bitapponwin64:true;  

    ```	
    g. 重點是這: 使用上一個From image container抓出剛剛build好的檔案，沒錯你必須先命名
    ```
    COPY --from=build /app/WebAPI/. ./inetpub/wwwroot
    ```	
    h. 再來是使用[Bruce](https://blog.kkbruce.net/)提供的HealthCheck方法去確保IIS都是ON的
    ```
    HEALTHCHECK --interval=30s `
    CMD powershell -command `
        try { `
        $response = iwr http://localhost/diagnostics -UseBasicParsing; `
        if ($response.StatusCode -eq 200) { return 0} `
        else {return 1}; `
        } catch { return 1 }

    ```	

4. 雷:
- 最後想說不自己動手做真的不知道哪裡有錯，尤其是前面我打算用volume的部分，很多斜線阿，雙引號阿，D槽吃不到啊，等等的問題。
- 版本最好先確認，1709跟1803兩個的版本是有很多差距的(建議用新的)
- Multi Stage用法，不使用這個會需要多好幾部動作
- 專案只丟需要的。建立.dockerignore檔案
- 逐步msbuild 專案
- Cmd或Powershell修改IIS方式
- 使用HealthCheck檢測Application是否正常(有可能container正常，但IIS掛)

5. Source: 
- [專案有Oracle安裝Docker deploy IIS 教學](http://www.codesin.net/post/Oracle_ODAC/) 
- 另一篇[教學](https://www.red-gate.com/simple-talk/sysadmin/virtualization/working-windows-containers-docker-running/)安裝，dockerfile deploy IIS (ServerCore)
- 安裝[教學](http://stevenfollis.com/2017/10/05/access-database-windows-container/)，修改IIS 32 bit

#### 結論

- 本篇用意在於初步認識Windows Container，但我想只是認識他好像沒啥用處，過一正子又會忘了，所以要能符合目前有用到的流程，總得加點什麼上去，所以才選擇網站佈版的流程，做個簡易CICD。不過目前還是建議先用在開發階段或是使用在POC或是Demo等其他專案做快速佈版用。這樣好處是可以乾淨一點，可以把這些範例專案放到dockerHub上。需要的時候在拉下來，NGINX，Redis，ELK，RabbiMQ等等的服務也可以這樣做，需要再拉就好。

```

```	
		

