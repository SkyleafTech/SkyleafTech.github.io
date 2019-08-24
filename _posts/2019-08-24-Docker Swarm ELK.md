---
layout:     post
title:      Docker ELK Part 4
subtitle:   Use docker Swarm to setup ELK
date:       2019-08-24
author:     BY Skyleaf
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Docker
    - ELK MultiNode
    - Docker Swarm
    - Visualizer
---
# Pre

> docker elk Setup Blog to record what I done these days: 
> 1. ELK stack Introduction，es single node setup by docker
> 2. ES multiNodes record
> 3. ELK Certificate record
> 4. ***Docker Swarm + ELK Setting***

![图](https://images.unsplash.com/photo-1564840726045-ab45d4db9140?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=401&q=80)

## Agenda

- Docker Swarm ELK Multi-node

# Docker Swarm

這邊一定要特別講到swarm，他是一個cluster，可以分散資料到各個node，可以橫向擴展，可以提高效能，不會單點fail。


## 實作

1. 透過docker-machine 建立三台vm
2. 在建立之前要先修改網卡，由於是Winodws Container底層是用Hyper-v來做的
   1. 一開始先改Hyper-v的"虛擬交換器管理員"設定IP4網址的轉換，官網也有講到如何更改 
   2. ![hypernet-ip](https://i.imgur.com/q6rZIox.png)
3. 建立vm
   1. docker-machine create -d hyperv --hyperv-virtual-switch "Primary Virtual Switch" vm1 
   2. ![manager-node](https://i.imgur.com/Mc8eyHv.png)
   3. 查看語法: docker-machine ls
4. 進入vm1 啟動swarm，取得master Token
   1. docker-machine ssh vm1 
   2. docker swarm init --advertise-addr 172.16.49.203 
   3. ![swarminit](https://i.imgur.com/Rof3inB.png)
5. 進入vm2跟vm3 加入vm01
   1. 接著去vm2跟vm3然後輸入在vm1拿到的Token 
   2. docker swarm join --token SWMTKN-1-3d81wqmfk0dp1drzk3fv1ozs5bm4049yu8k20dofqlin73pyhh-2dk12n91pyodwgu02fof5ktd1 172.16.49.203:2377 
6. 檢查目前cluster node
   1. docker node ls
   2. ![node]https://i.imgur.com/OtKyuCk.png)

7. 每台vm都記得開啟max_map_count上限給262144
   - sudo sysctl -w vm.max_map_count=262144 
8.  注意以下操作都是在master node執行
9.  複製docker-compose.yml到vm01上
    1. 可以透過ssh工具，也可以透過docker語法
    2. 使用ssh進入vm1的話，由於hyper-v 使用的VM是boot2docker，SSH的default是帳密是docker/tcuser，[boot2docker](https://github.com/boot2docker/boot2docker/blob/master/README.md)也有註明
    3. 使用docker語法的話，Docker-machine scp，[scp](https://docs.docker.com/machine/reference/scp/) 
       - docker-machine scp docker-compose.yml myvm1:~ 
10. 透過stack執行部屬 docker-compose-xxxx.yml
    1.  docker stack deploy -c docker-stack.yml elk 
    2.  docker stack deploy --compose-file docker-compose-cluster.yml elk 
    3.  ![dockerSwarm](https://i.imgur.com/k9lU2q4.png)
11. 查看node的腳色，先透過docker node ls抓出node的Id 
    1.  docker node inspect NodeId 
    2.  ![inspect](https://i.imgur.com/ER6baVY.png)
12. 查看是否有錯誤 
    1.  Docker service ls  
    2.  查看每個service的佈署狀態 
    3.  Docker service ps elk_kibana 
    4.  發現我沒把kibana的folder丟上去vm1內，重新deploy後發現docker ps 就會出現服務了
    5.  ![checkKibana](https://i.imgur.com/X5DjQk3.png)
13. 透過Deploy語法指定ELK Service的部屬
    -  放在這Github
14. 最終部屬架構
    - ![swarmelk](https://i.imgur.com/dKVo3bg.png)
15. 移除語法
    -  docker stack rm elk
16. Rolling update 
    -  docker service update --replicas=3  elk_elasticsearch
17. 離開swarm 
    - docker swarm leave --force 


## GUI 套建 Docker Swarm Visualizer 

- 主要用來查看部屬而已

### 安裝步驟

    1. docker run -it --restart always -d -p 5000:8080 -v /var/run/docker.sock:/var/run/docker.sock dockersamples/visualizer 
    2. 必須要在vm1裡面執行安裝語法 
    3. docker service create --name=viz --publish=5000:8080/tcp --constraint=node.role==manager --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock dockersamples/visualizer 
    4. 在vm1內透過docker service ls 去抓出IP addr如果你忘了的話 
    5. http://172.16.49.203:5000/ 
    6. 如果是docker service建立的container，會需要用docker service rm來移除，如果只是透過docker rm xxx，它會自動重建
    7. 刪除語法 always restart 
       1. sudo docker update --restart=no 
       2. Docker rm XXX 



# Consulsion

最後終於結束這四篇的安裝，歡迎大家可以嘗試嘗試。有些東西不試就不會知道當中的眉眉角角。這部前應該是可以直接使用的，如果要Production還要依據參數做一點修改，另外可以安裝Nginx跟Curator。一個是反向代理，一個是定時刪除es indies。差不多就這樣，最近公司系統好像出狀況了，該回歸正途了，感謝觀看。



## 來源

1. [**Elastic**](https://www.elastic.co/what-is/elk-stack)
2. [**Docker-Swarm-Beginners-Guide**](https://github.com/twtrubiks/docker-swarm-tutorial)
3. [**Learning ELK Stack**](https://www.amazon.com/Learning-ELK-Stack-Saurabh-Chhajed/dp/1785887157)
4. [**elk architecture**](https://www.atechref.com/blog/elk/elk-stack-architecture/)
5. [elastic/stack-docker](https://github.com/elastic/stack-docker)
6. [configuring-tls-docker](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/configuring-tls-docker.html)
7. [scp](https://docs.docker.com/machine/reference/scp/)
8. [boot2docker](https://github.com/boot2docker/boot2docker/blob/master/README.md)


