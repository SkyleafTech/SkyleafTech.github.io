---
layout:     post
title:      Docker ELK Part 1
subtitle:   Use docker to setup ELK single node
date:       2019-08-01
author:     BY Skyleaf
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Docker
    - ELK single node
---
# 前言

> 之後會有一系列 docker elk 建置文章來記錄一下這正子的實作: 
> 1. ***ES single node setup by docker***
> 2. ES multiNodes的紀錄
> 3. 驗證的機制(certificate)的紀錄
> 4. Docker Swarm的ELK設定

![图](https://images.unsplash.com/photo-1564591167348-6a45f42dc223?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=80)

## Agenda

- 心路歷程
- Docker ELK

# 心路歷程

原本Elasticsearch就是一個蠻流行的搜尋工具，以前還是2.0版的時候使用過，當時還自建了Vue的頁面來呈現搜尋結果，當時只使用了logstash跟elasticsearch來做資料搜尋。個人覺得這個開源的產品真的很屌，速度很快，50萬筆的資料10分鐘左右就load進去了，而且每筆的查詢也是蠻快的，但資料大跟多當然會影響效能，但也是目前技術上可以接受的。在排行榜上也是名列前茅。[排名連結](https://db-engines.com/en/ranking/search+engine)。


4年過去了，又要再次接觸ELK，但這次不像之前那樣建置了，以前是放在windows server上，安裝java，安裝很多套件，還自己硬刻了一個GUI介面。現在有x-pack了(要錢)，整合了幾乎所有的plugin套件。以前還要用Maven去安裝，現在都不用了。現在ELK的版本是7.x了，加了x-pack功能後連名稱都改成ELK Stack了，公司好像也改成elastic。不過這次要挑戰的不只是ELK stack，還要挑戰的是把它安裝在CentOS上，而且是透過Docker，自從開源這幾年瘋狂盛行，微軟也開放擁抱linux後，可以說未來不使用到Linux的機率是0，除非你想當守門員，那就比較例外了，但身為科技人一定要不斷更新，不斷學習。還記得第一次遇到linux系統困境時，是因為要架設Redis HA，當年Redis HA的資訊除了官網還真是稀少，不過現在多的跟山一樣。那時候我奇葩的使用了windows版的Redis來做實現，再丟給Infra team幫我建置，來來去去搞了一正子後才成功。後來架設RabbitMQ又遇到一次，當時我就覺得不能再逃避了，接著Docker走紅了，K8s起來了，這時候不使用command line就很遜的感覺。但是以前排斥linux就是因為command line，再來是安裝VM麻煩，網路設定麻煩，本機電腦資源不足，安裝玩VM想測cluster超慢等等的問題，讓我相當挫折。當時連要copy一個檔案從linux回到本機我都不知道怎弄，問了同事也不說怎用，吱吱嗚嗚的，對處處碰壁，我只能說是環境逼我們學習了，林北自己來學。學一套不需要開VM的不需要麻煩其他人的技術，ㄟ!!!那不就是docker。

# Docker ELK 介紹

這次打算用Docker 實作ELK stack來做log的搜尋，預計會有**ES單node**，跟**ES multiNodes的紀錄**，再來是**驗證的機制(certificate)的紀錄**，最後是搭配**Docker Swarm的ELK設定**，種共會有四篇。然後這次的ELK設計上會包含Filebeats的架構。全部都是透過指令去建置，有點像是infrastructure as code的概念，兩三個指定就幫你建好整套ELK。你可能會問為什麼使用docker swarm而不是K8s? 因為我對K8s不熟，XD，技術不是呼呼口號阿，老闆，我知道他紅，弄個POC給你聞香還可以，但你想把它當作Best Practice又不事前訓練，以為它自己會好，接手的工程師們都應該要會，這思維我不懂，沒培養他們，難道是要觀落陰來得到的技能嗎? 要熟悉K8s運作，你最少需要linux基礎，command line的操作基礎，Docker概念，Cluster概念，加上這次的ELK設定跟它的基本使用。大聲呼喊: 老闆，我們知道要K8s，但是具體，具體該怎麼做啊? (立委再次翻白眼)!! XD 不聊政治。


## ELK Stask架構: 
1. Filebeat service偵測到實體xxx.log檔案異動後，會打向logstash server
2. logstash service收到資料後，會解析log資料內容，轉成filter內定義好的格式到elasticsearch server
3. elasticsearch收到資料會拆分成不同的shards，分散到不同的es node上
4. Kibana 用來檢視，查看這些log資料的內容，還可以透過一些dashboard產出不同的表格，以利分析


![elk single node architecture](https://i.imgur.com/K5poOPg.png)


### Filebeat

首先[下載Filebeat](https://www.elastic.co/downloads/)，filebeat因為架構上我會把它放到各自的Server上，以目前都是Windows Server的專案來看，是需要手動先安裝這個，當然這是因各位的專案架構而會有所不同。順便說道為什麼會使用這個，因為以前我們都是手動或是自動mount資料夾到某個共用server上，再把包丟到logstash機器上讓logstash讀實體log檔案，常常遇到的問題是log實體檔案過大，透過這個就可以把分布在10幾台以上的log透過filebeat自己打資料到logstash機器上。算是有解決道問題，所以特別把這個也加進來，整體文章結構比較完整一點。

這安裝很簡單，下載完後，解壓縮放到你要的資料夾，在執行filebeat.exe檔案即可，也可以掛載成Windows Service。啟動前可以先修改filebeat.yml檔案指定你要去的logstash位置跟log寫入的格式，以下是最基礎的配置:
    
    ```
        - input_type: log
            paths:
                - C:\MyWeb\Logs\201908\*
        - output.logstash:
            hosts: ["localhost:5044"]

        logging.to_files: true
        logging.files:
    ```


### ELK docker-Compose檔


環境: 

- Windows 10
- docker desktop 我是下載最新的2.1.0
- docker Engine 19.03跟Compose:1.24.1 這些都是docker一並幫你裝到好
- ELK Stack採用6.8.1

稍微註明一下，docker的基礎用法我就不再這裡介紹了，linux的基礎也是，cluster等等的概念也都不再這裡講述，因為蠻多文章都有在介紹，沒必要重新蓋輪子了，有的寫得也不錯，再請同學們自行google吧，推薦可以查看[沈大大的](https://github.com/twtrubiks/docker-elk-tutorial)，再來是使用docker compose 建置ELK。通常我們會寫dockerfile來建置每個服務，但是當我們有多個服務，我們可以透過docker-compose.yml檔案把它們串在一起，寫法跟dockerfile差不多。


docker-compose.yml: 
        
    ```
        version: '3'

        services:
        elasticsearch:
            image: elasticsearch:6.8.1
            container_name: elasticsearch
            environment:
            cluster.name: 'elastic-cluster'
            node.name: 'elasticsearch'
            node.master: 'true'
            node.data: 'true'
            bootstrap.memory_lock: 'true'
            xpack.license.self_generated.type: 'basic'
            #Discovery
            discovery.zen.ping_timeout: '30s'
            discovery.zen.minimum_master_nodes: '1'
            gateway.recover_after_data_nodes: '1'
            thread_pool.bulk.queue_size: '3000'
            indices.breaker.request.limit: '10%'
            search.default_search_timeout: '30s'
            indices.fielddata.cache.size:  '30%'

            ES_JAVA_OPTS: '-Xms512m -Xmx512m'
            xpack.security.enabled: 'false'
            xpack.monitoring.enabled: 'true'
            xpack.graph.enabled: 'true'
            xpack.watcher.enabled: 'true'
            
            ulimits:
            memlock:
                soft: -1
                hard: -1
            volumes:
            - ./esdata01:/usr/share/elasticsearch/data
            ports:
            - 9200:9200
            networks:
            - docker_elk
        
        logstash:
        image: logstash:6.8.1
        volumes:
            - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
        ports:
            - 5044:5044
            - "12201:12201/udp"
        environment:
            http.host: '0.0.0.0'
            pipeline.workers: '12'
            pipeline.batch.size: '30000'
            pipeline.batch.delay: '50'
            config.reload.automatic: 'true'
            config.reload.interval: '10s'
            xpack.monitoring.enabled: 'true'
            LS_JAVA_OPTS: "-Xmx256m -Xms256m"
        networks:
            - docker_elk
        depends_on:
            - elasticsearch
        
        kibana:
            image: kibana:6.8.1
            environment:
            server.host: '0.0.0.0'
            server.name: 'kibana'
            ELASTICSEARCH_URL: 'http://elasticsearch:9200'
            xpack.security.enabled: 'false'
            xpack.monitoring.enabled: 'true'
            xpack.monitoring.kibana.collection.enabled: 'true'
            xpack.graph.enabled: 'true'
            xpack.reporting.enabled: 'true'
            xpack.reporting.encryptionKey: 'EnCryptNoChangeMe'
            xpack.reporting.kibanaServer.hostname: 'kibana'
            ports:
            - 5601:5601
            networks:
            - docker_elk
            depends_on:
            - elasticsearch

        networks:
        docker_elk:

    ```




# 結論

這篇是個開端，內容其實還好，都蠻基本的，不過後面就是我真正努力花時間的地方了，主要除了給需要的朋友，再來也是我自己的紀錄。





## 來源

1. [**Elastic**](https://www.elastic.co/what-is/elk-stack)
2. [**docker-elk-tutorial**](https://github.com/twtrubiks/docker-elk-tutorial)
3. [**Learning ELK Stack**](https://www.amazon.com/Learning-ELK-Stack-Saurabh-Chhajed/dp/1785887157)
4. [**elk architecture**](https://www.atechref.com/blog/elk/elk-stack-architecture/)



