---
layout:     post
title:      Docker ELK Part 2
subtitle:   Use docker to setup ELK MultiNode
date:       2019-08-08
author:     BY Skyleaf
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Docker
    - ELK MultiNode
---
# 前言

> docker elk建置文章來記錄一下這正子的實作: 
> 1. ELK stack介紹，es single node setup by docker
> 2. ***ES multiNodes的紀錄***
> 3. 驗證的機制(certificate)的紀錄
> 4. Docker Swarm的ELK設定

![图](https://images.unsplash.com/photo-1564840726045-ab45d4db9140?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=601&q=80)

## Agenda

- Elasticsearch Muli-Node

# Elasticsearch Muli-Node

其實上一篇雖然說要介紹ELK Stask，但好像沒介紹什麼細節，只是帶過我的心路歷程比較多，XD。這篇我打算講一下yml檔案內的設定。設定方式其實百百種，有的會透過build參數去觸發docker file，有的environment的環境變數會統一寫在docker-compose.yml檔案內，其實都可以實現，正式機器我會建議分開設定來實現，而範例或是POC等等測試環境我就會建議寫成一整個。理由是好管理。分開設定的意思是，例如Logstash.yml可以寫在docker-compose.yml的environment欄位內，讓他覆蓋container內的參數，這樣統一寫在一起。而分開的方式就是透過volume方式去指定另一個檔案，或是source去指定target檔案。

接下來直接看設定範本，以下是Elasticsearch兩個node的cluster設定，搭配2個logstash，1台kibana的yml檔案:

ˋˋˋ
    version: '3'

    services:
    elasticsearch:
        image: elasticsearch:6.8.1
        container_name: elasticsearch
        environment:
        cluster.name: 'elastic-cluster'
        node.name: 'elasticsearch'
        node.master: 'true'
        node.data: 'false'
        node.ingest: 'false'
        bootstrap.memory_lock: 'true'
        xpack.license.self_generated.type: 'basic'
        ES_JAVA_OPTS: '-Xms512m -Xmx512m'
        xpack.security.enabled: 'false'
        xpack.monitoring.enabled: 'true'
        xpack.graph.enabled: 'false'
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
    es02:
        image: elasticsearch:6.8.1
        container_name: es02
        environment:
        cluster.name: 'elastic-cluster'
        node.name: 'es02'
        node.master: 'false'
        node.data: 'true'
        
        bootstrap.memory_lock: 'true'
        discovery.zen.ping.unicast.hosts: 'elasticsearch,es03'
        xpack.license.self_generated.type: 'basic'
        ES_JAVA_OPTS: '-Xms512m -Xmx512m'
        xpack.security.enabled: 'false'
        xpack.monitoring.enabled: 'true'
        xpack.graph.enabled: 'false'
        xpack.watcher.enabled: 'true'
        ulimits:
        memlock:
            soft: -1
            hard: -1
        volumes:
        - ./esdata02:/usr/share/elasticsearch/data
        networks:
        - docker_elk
    es03:
        image: elasticsearch:6.8.1
        container_name: es03
        environment:
        cluster.name: 'elastic-cluster'
        node.name: 'es03'
        node.master: 'false'
        node.data: 'true'
        
        bootstrap.memory_lock: 'true'
        discovery.zen.ping.unicast.hosts: 'elasticsearch,es02'
        xpack.license.self_generated.type: 'basic'
        ES_JAVA_OPTS: '-Xms512m -Xmx512m'
        xpack.security.enabled: 'false'
        xpack.monitoring.enabled: 'true'
        xpack.graph.enabled: 'false'
        xpack.watcher.enabled: 'true'
        ulimits:
            memlock:
                soft: -1
                hard: -1
        volumes:
        - ./esdata03:/usr/share/elasticsearch/data
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
        #path.config: '/usr/share/logstash/pipeline'
        xpack.monitoring.enabled: 'true'
        LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
        - docker_elk
    depends_on:
        - elasticsearch
    logstash2:
        image: logstash:6.8.1
        volumes:
        - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
        ports:
        - "5043:5043"
        environment:
            http.host: '0.0.0.0'
            xpack.monitoring.enabled: 'true'
            LS_JAVA_OPTS: '-Xmx256m -Xms256m'
        networks:
        - docker_elk
        depends_on:
        - elasticsearch
    kibana:
        image: kibana:6.8.1
        environment:
        xpack.security.enabled: 'false'
        xpack.monitoring.enabled: 'true'
        server.name: 'kibana'
        server.host: '0.0.0.0'
        ELASTICSEARCH_URL: 'http://elasticsearch:9200'
        ports:
        - 5601:5601
        networks:
        - docker_elk
        depends_on:
        - elasticsearch

    networks:
    docker_elk:

ˋˋˋ

## 設定上解釋

1. Elasticsearch

   - 使用docker-compose.yml version 3 版本，你說這有差嗎?答案是當然有，有先語法只支援到某些版本就停用了，有些則是更高的版本才有支援，詳情可查看[官網](https://docs.docker.com/compose/compose-file/)
   - elasticsearch 的image我使用6.8.1的，這裡發現，一開始用在實作的時候使用的都是[elk stack官網的docker設定](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/docker.html)，所以使用的都是elastic公司的docker hub而不是docker public hub上的image，後來統一docker public hub的，名稱短一點:
     - elastic公司的: docker.elastic.co/elasticsearch/elasticsearch:6.8.1
     - docker public hub: elasticsearch:6.8.1

   - 指定cluster-name，這再其他的es node一定要依一樣
   - node-master給你要的es master node腳色
   - node-data給你要存data的腳色
   - 還有類似Tribe node跟ingest node腳色，ingest node是對document做預先處理，
   - 以前還有client node這個腳色，但後來我發現沒了，貌似新版都統一Coordinating node，把client node開除了!!以前是用在分發請求跟合併結果的。每個節點都有其特色，而且怎樣調整到最好的狀態會取決於你存data的方式，哪個cpu要高搭配哪個腳色，哪個memory高搭配什麼腳色等等。
   - 這邊建議統一先決定一個版本來實作先，因為每個版本的參數都不一樣，實際上影響了什麼都得查看文件，這幾年我覺得異動的蠻大的，跟我以前認知都有點差距了，甚至有時候文件只顯示用這個，但是卻沒解釋的窘境，反正有用到在查吧!
   - bootstrap.memory_lock 這個參數我要說一下，這個系統做swapping有關，我這邊遇到的狀況可能解釋不夠精細還請參考就好，我在系統初步不夠的時候會先把他關閉，不然我elk multi-node根本起不來，但官方事件是你先預留這個記憶體量給你的elk系統，所以當我開啟的時候我設定了ulimits，也就是使用限制，不過在swarm模式我又遇到困難了(要改設定在linux OS上)。

   ˋˋˋ
       ulimits:
           memlock:
               soft: -1
               hard: -1

   ˋˋˋ
   - xpack.license.self_generated.type: 這是就是x-pack試用版本設定，basic或是trial，可以使用一個月，basic我猜功能少一點
   - ES_JAVA_OPTS: 這跟memory heap有關
   - xpack.security.enable: 這是x-pack內的套件功能了，裡面有蠻多的我就不都解釋，這個是是否要密碼才能使用，下一篇會講設定密碼certificate
   - xpack.monitoring.enabled: 這是可以讓你在kibana的monitor看到他的數據資料
   - volumes: 這就是掛載我本機的資料夾給container內的elasticsearch的shards儲存用，因為一般container關閉後會清空所有東西，但我想保留之前就在elasticsearch內的資料，就透過這是方式
   - ports: 我給的是官方的9200
   - networks: 我指定docker_elk
   - 其餘es02跟es03的設定差異在於
     - node.name: 名稱當然不一樣
     - node.master跟node.data: 腳色上看你要的定位，es02跟es03的我給她data腳色，elasticsearch我給他master腳色

2. Logstash

   - images: 使用logstash:6.8.1
   - volumes: 蠻重要的，我本機先寫好在映射到container上的logstash的位置上，這邊只有映射pipline的，也就是input，output的設定，沒有filter那些
   - ports: 5044，跟官方的一樣
   - environment:
     - http.host: 其實default就是0.0.0.0，這邊須注意他是否真的有apply到logstash上，在swarm環境上寫法是不同的
     - xpack.monitoring.enabled: 讓他在kibana開啟monitor查看數據
   - depends_on: 這個是代表他會先執行elasticsearch，好了後才執行自己
   - 第二個logstash給他命名logstash2
     - 設定上大同小異，只是port要給不同，這裡給5043

3. Kibana

   - images使用kibana:6.8.1
   - environment:
     - 開啟monitoring
     - server.name: 就給他kibana吧
     - server.host: 都設定本機先
     - ELASTICSEARCH_URL: 這個最重要，因為要指向哪個elasticsearch位置，而且你看寫法不同，我試過elasticseatch.url會失敗，有再懷疑是設定format要改成底線，domain name則是可以指定container_name，因為他們在同樣的網路上
     - ports: 5601


4. Linux環境變數設定

   - 官方建議設定max_map_count參數最低要 262144
   - 先查看是否有設定
     - $ grep vm.max_map_count /etc/sysctl.conf
   - Windoes Container透過doker-machine ssh進去後設定 
     - sudo sysctl -w vm.max_map_count=262144


5. 實測:

    - docker-compose up -d 就可以啟動他了
    - 剛開始的時候你會遇到很多狀況，會需要查看log來發現問題，可能是打錯字，可能是位置設定不對等等，可以善用這兩個工具，另外建議如果遇到狀況可以先啟elasticeasach就好，確保elasticsearch是起來的先
    - 這邊建議大家可以先裝[portainer](https://github.com/portainer/portainer)，來查看container的狀態，跟查看log，很多人會跟你說先不要用，但是我覺得超好用，有其是查看log
      - $ docker volume create portainer_data
      - $ docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
    - 另一個工具是[Cerebro](https://github.com/lmenezes/cerebro)，他是已前的kpof，他可以查看elasticsearch狀態，也可以直接操控elasticsearch喔，建議設密碼，不然被刪了就產了
      - docker run -p 9000:9000 --env-file env-ldap  lmenezes/cerebro

# 結論

以上大概就是這樣了，歡迎有需要的朋友嘗試看看。



## 來源

1. [**Elastic**](https://www.elastic.co/what-is/elk-stack)
2. [**docker-elk-tutorial**](https://github.com/twtrubiks/docker-elk-tutorial)
3. [**Learning ELK Stack**](https://www.amazon.com/Learning-ELK-Stack-Saurabh-Chhajed/dp/1785887157)
4. [**elk architecture**](https://www.atechref.com/blog/elk/elk-stack-architecture/)



