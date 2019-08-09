---
layout:     post
title:      Docker ELK Part 3
subtitle:   Use docker to setup ELK certificate
date:       2019-08-09
author:     BY Skyleaf
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Docker
    - ELK MultiNode
---
# Pre

> docker elk Setup Blog to record what I done these days: 
> 1. ELK stack Introduction，es single node setup by docker
> 2. ES multiNodes record
> 3. ***ELK Certificate record***
> 4. Docker Swarm + ELK Setting

![图](https://images.unsplash.com/photo-1564840726045-ab45d4db9140?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=601&q=80)

## Agenda

- Elasticsearch Certificate

# ELK Certificate

On this post I have encountered a difficulty, "Where can I start with it", but eventually I found these two blog, funny thing is the one only cover ***es multi node ssl transport communicating***, another one cover ***single node of elasticsearch + logstash + kibana***, but not the es multi node with kibana and logstash; therefore I decided to combine it together.

Here is the two sources: [configuring-tls-docker](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/configuring-tls-docker.html)，[official elastic github](https://github.com/elastic/stack-docker/blob/master/docker-compose.yml)。

These two example does give me a lecture of how the docker can create certificate for elasticsearch and how to use docker compose to set it up for entire elk with authentication. Normally, we use elasticsearch-certgen tool to generate the certificate, then unzip and copy the crt and key file to our elk folder, ( we use instances.yml to generate all serivces's certificates file at a time.). However this time we want to do it in Docker. How do we gernerate certificate and also setup password to all elk service?



## Generate Certificate 

### First Trial:

When I struggling on this question, I have discoverd the first example the [configuring-tls-docker](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/configuring-tls-docker.html)，I realized I need two steps to make it happen. Firstly, we need to run the create-certs.yml to generate the certificates, then I can run docker-compose.yml file to create es multi node service. But somehow I failed with this example, I'm confused with the password and cetificate, I tried to enter http://127.0.0.1:9200 with the password that I put in .env file, but it does not work. Dont know why??? But I can re-generate it by the following code，which will generate the elasticsearch password for me. But this is not what I want.

ˋˋˋ
    docker exec es01 /bin/bash -c "bin/elasticsearch-setup-passwords \
    auto --batch \
    -Expack.ssl.certificate=certificates/es01/es01.crt \
    -Expack.ssl.certificate_authorities=certificates/ca/ca.crt \
    -Expack.ssl.key=certificates/es01/es01.key \
    --url https://localhost:9200"
ˋˋˋ

### Second Trial: 

I have discoverd the second example the  [elastic/stack-docker](https://github.com/elastic/stack-docker). We need docker-compose file and shell files to do it, and find out we need to do more work then just generate certificate and make it work.


Firstly, we run ***docker-compose -f setup.yml up***. To trigger setup.yml and generate certificate and set ELASTIC_PASSWORD to elasticseatch service, after generate certificate files and keystores, copy the certificate files to each services. Secondly, we can run ***docker-compose up -d***, to start up the elk serivces, and try to login to http://localhost:5601 with username: elastic, password will print out on the console when you first run the setup.yml. Yes, this does works!! Thanks God.

#### Steps: 

1. Pre request:  
   1. open powershell and run: $Env:COMPOSE_CONVERT_WINDOWS_PATHS=1 
   2. then at environment path add following
   3. PWD = /d/Docker/ELK/stack-docker-master 
   4. ![set Environment](https://i.imgur.com/FfVkBGp.png)
2. Create certificate files，setup keystore 
   1. docker-compose -f setup.yml up 
   2. at end output print elastic password，need to remeber it, you can change it after login to kibana  
3. Start up ELK services
   1. docker-compose up –d 

## Combine with multi es nodes

After the [elastic/stack-docker](https://github.com/elastic/stack-docker), we know how to generate certificate. In this part we will add two elasticsearch data node in it, I add two more elasticsearch services on docker-compose file, basiclly just copy the elasticseatch service's setting, and rename the service to es02 and es03, also change the node.name, node.master, node.data, discovery.zen.ping.unicast.hosts。To enable authenticate we should add xpack.security.enabled to true. **security.transport.ssl** setting is for es multi node to communicate, and we need to set the xpack.ssl.* and xpack.security.http to load the certificate.

![configWilCetificate](https://i.imgur.com/xqdXsiX.png)

The tricky part is I have add both setup_es02 and setup_es03 services in docker compose file, and create setup-es02.sh and setup-es03.sh. These files will generate keystores and set $ELASTIC_PASSWORD. Of course you can do it in a setup-xxx.sh files. Another thing is I move environment setting for both Logstash and Kibana, because I find out the format is not right, normally we use period to seperate naming but somehow it has be underscore. Ex: elasticsearch_url instead of elasticsearch.url。 Also I set the pwd path in .env file, not in system environment's path. When I run setup_Kibana I set depend_on for not just elasticsearch also the es02 and es03 services, so these es nodes will start up and set the ELASTIC_PASSWORD before Kibana connect to it。

![structure](https://i.imgur.com/8dwEXl7.png)

That's check the yml files: 

ˋˋˋ

    version: '3.6'

    services:
    elasticsearch:
        image: elasticsearch:${TAG}
        container_name: elasticsearch
        secrets:
        - source: ca.crt
            target: /usr/share/elasticsearch/config/certs/ca/ca.crt
        - source: elasticsearch.keystore
            target: /usr/share/elasticsearch/config/elasticsearch.keystore
        - source: elasticsearch.key
            target: /usr/share/elasticsearch/config/certs/elasticsearch/elasticsearch.key
        - source: elasticsearch.crt
            target: /usr/share/elasticsearch/config/certs/elasticsearch/elasticsearch.crt
        environment:
        cluster.name: 'elastic-cluster'
        node.name: 'elasticsearch'
        node.master: 'true'
        node.data: 'true'
        #node.ingest: 'false'
        bootstrap.memory_lock: 'true'
        xpack.license.self_generated.type: 'trial'
        # #Discovery
        discovery.zen.ping.unicast.hosts: 'es02, es03'
        discovery.zen.ping_timeout: '30s'
        discovery.zen.minimum_master_nodes: '1'
        gateway.recover_after_data_nodes: '1'

        ES_JAVA_OPTS: '-Xms512m -Xmx512m'
        xpack.security.enabled: 'true'
        xpack.monitoring.enabled: 'true'
        xpack.graph.enabled: 'true'
        xpack.watcher.enabled: 'true'
        xpack.monitoring.collection.enabled: 'false'
        #password set up
        network.host: '0.0.0.0'
        transport.host: '0.0.0.0'
        xpack.security.http.ssl.enabled: 'true'
        xpack.security.http.ssl.verification_mode: 'certificate'
        xpack.security.http.ssl.key:  'certs/elasticsearch/elasticsearch.key'
        xpack.security.http.ssl.certificate: 'certs/elasticsearch/elasticsearch.crt'
        xpack.security.http.ssl.certificate_authorities: 'certs/ca/ca.crt'

        xpack.security.transport.ssl.enabled: 'true'
        xpack.security.transport.ssl.key:  'certs/elasticsearch/elasticsearch.key'
        xpack.security.transport.ssl.certificate: 'certs/elasticsearch/elasticsearch.crt'
        xpack.security.transport.ssl.certificate_authorities: 'certs/ca/ca.crt'
        ELASTIC_PASSWORD: $ELASTIC_PASSWORD
        xpack.security.transport.ssl.verification_mode: 'certificate'
        xpack.ssl.certificate: 'certs/elasticsearch/elasticsearch.crt'
        xpack.ssl.key: 'certs/elasticsearch/elasticsearch.key'
        xpack.ssl.certificate_authorities: 'certs/ca/ca.crt'
        
        ulimits:
        memlock:
            soft: -1
            hard: -1
        volumes:
        - ./esdata01:/usr/share/elasticsearch/data
        - './scripts/setup-users.sh:/usr/local/bin/setup-users.sh:ro'
        ports:
        - 9200:9200
        - 9300:9300
        networks:
        - docker_elk
        healthcheck:
        test: curl --cacert /usr/share/elasticsearch/config/certs/ca/ca.crt -s https://localhost:9200 >/dev/null; if [[ $$? == 52 ]]; then echo 0; else echo 1; fi
        interval: 30s
        timeout: 10s
        retries: 5

    es02:
        image: elasticsearch:${TAG}
        container_name: es02
        secrets:
        - source: ca.crt
            target: /usr/share/elasticsearch/config/certs/ca/ca.crt
        - source: es02.keystore
            target: /usr/share/elasticsearch/config/es02.keystore
        - source: es02.key
            target: /usr/share/elasticsearch/config/certs/es02/es02.key
        - source: es02.crt
            target: /usr/share/elasticsearch/config/certs/es02/es02.crt
        environment:
        cluster.name: 'elastic-cluster'
        node.name: 'es02'
        node.master: 'false'
        node.data: 'true'
        bootstrap.memory_lock: 'true'
        xpack.license.self_generated.type: 'trial'
        #Discovery
        
        discovery.zen.ping.unicast.hosts: 'elasticsearch, es03'
        discovery.zen.ping_timeout: '30s'
        discovery.zen.minimum_master_nodes: '1'
        gateway.recover_after_data_nodes: '2'

        ES_JAVA_OPTS: '-Xms512m -Xmx512m'
        xpack.security.enabled: 'true'
        xpack.monitoring.enabled: 'true'
        xpack.graph.enabled: 'true'
        xpack.watcher.enabled: 'true'
        xpack.monitoring.collection.enabled: 'true'
        #password set up
        network.host: '0.0.0.0'
        transport.host: '0.0.0.0'
        xpack.security.http.ssl.enabled: 'true'
        xpack.security.http.ssl.verification_mode: 'certificate'
        xpack.security.http.ssl.key:  'certs/es02/es02.key'
        xpack.security.http.ssl.certificate: 'certs/es02/es02.crt'
        xpack.security.http.ssl.certificate_authorities: 'certs/ca/ca.crt'

        xpack.security.transport.ssl.enabled: 'true'
        xpack.security.transport.ssl.key:  'certs/es02/es02.key'
        xpack.security.transport.ssl.certificate: 'certs/es02/es02.crt'
        xpack.security.transport.ssl.certificate_authorities: 'certs/ca/ca.crt'
        ELASTIC_PASSWORD: $ELASTIC_PASSWORD
        xpack.security.transport.ssl.verification_mode: 'certificate'
        xpack.ssl.certificate: 'certs/es02/es02.crt'
        xpack.ssl.key: 'certs/es02/es02.key'
        xpack.ssl.certificate_authorities: 'certs/ca/ca.crt'
        ulimits:
        memlock:
            soft: -1
            hard: -1
        volumes:
        - ./esdata02:/usr/share/elasticsearch/data
        - './scripts/setup-users.sh:/usr/local/bin/setup-users.sh:ro'
        networks:
        - docker_elk


    es03:
        image: elasticsearch:${TAG}
        container_name: es03
        secrets:
        - source: ca.crt
            target: /usr/share/elasticsearch/config/certs/ca/ca.crt
        - source: es03.keystore
            target: /usr/share/elasticsearch/config/es03.keystore
        - source: es03.key
            target: /usr/share/elasticsearch/config/certs/es03/es03.key
        - source: es03.crt
            target: /usr/share/elasticsearch/config/certs/es03/es03.crt
        environment:
        cluster.name: 'elastic-cluster'
        node.name: 'es03'
        node.master: 'false'
        node.data: 'true'
        bootstrap.memory_lock: 'true'
        xpack.license.self_generated.type: 'trial'
        #Discovery
        discovery.zen.ping.unicast.hosts: 'elasticsearch, es02'
        discovery.zen.ping_timeout: '30s'
        discovery.zen.minimum_master_nodes: '1'
        gateway.recover_after_data_nodes: '2'
        thread_pool.bulk.queue_size: '3000'
        indices.breaker.request.limit: '10%'
        search.default_search_timeout: '30s'
        indices.fielddata.cache.size:  '30%'

        ES_JAVA_OPTS: '-Xms512m -Xmx512m'
        xpack.security.enabled: 'true'
        xpack.monitoring.enabled: 'true'
        xpack.graph.enabled: 'true'
        xpack.watcher.enabled: 'true'
        xpack.monitoring.collection.enabled: 'true'
        #password set up
        network.host: '0.0.0.0'
        transport.host: '0.0.0.0'
        xpack.security.http.ssl.enabled: 'true'
        xpack.security.http.ssl.verification_mode: 'certificate'
        xpack.security.http.ssl.key:  'certs/es03/es03.key'
        xpack.security.http.ssl.certificate: 'certs/es03/es03.crt'
        xpack.security.http.ssl.certificate_authorities: 'certs/ca/ca.crt'

        xpack.security.transport.ssl.enabled: 'true'
        xpack.security.transport.ssl.key:  'certs/es03/es03.key'
        xpack.security.transport.ssl.certificate: 'certs/es03/es03.crt'
        xpack.security.transport.ssl.certificate_authorities: 'certs/ca/ca.crt'
        ELASTIC_PASSWORD: $ELASTIC_PASSWORD
        xpack.security.transport.ssl.verification_mode: 'certificate'
        xpack.ssl.certificate: 'certs/es03/es03.crt'
        xpack.ssl.key: 'certs/es03/es03.key'
        xpack.ssl.certificate_authorities: 'certs/ca/ca.crt'
        ulimits:
        memlock:
            soft: -1
            hard: -1
        volumes:
        - ./esdata03:/usr/share/elasticsearch/data
        - './scripts/setup-users.sh:/usr/local/bin/setup-users.sh:ro'
        networks:
        - docker_elk

    
    logstash:
        image: logstash:${TAG}
        secrets:
        - source: logstash.conf
            target: /usr/share/logstash/pipeline/logstash.conf
        - source: logstash.yml
            target: /usr/share/logstash/config/logstash.yml
        - source: logstash.keystore
            target: /usr/share/logstash/config/logstash.keystore
        - source: ca.crt
            target: /usr/share/logstash/config/certs/ca/ca.crt
    
        ports:
        - 5044:5044
        - "12201:12201/udp"
    
        networks:
        - docker_elk
        depends_on:
            - elasticsearch
        healthcheck:
        test: bin/logstash -t
        interval: 60s
        timeout: 50s
        retries: 5
    
    kibana:
        image: kibana:${TAG}
        secrets:
        - source: kibana.yml
            target: /usr/share/kibana/config/kibana.yml
        - source: kibana.keystore
            target: /usr/share/kibana/data/kibana.keystore
        - source: ca.crt
            target: /usr/share/kibana/config/certs/ca/ca.crt
        - source: kibana.key
            target: /usr/share/kibana/config/certs/kibana/kibana.key
        - source: kibana.crt
            target: /usr/share/kibana/config/certs/kibana/kibana.crt
        
        ports:
        - 5601:5601
        networks:
        - docker_elk
        depends_on:
        - elasticsearch
        healthcheck:
        test: curl --cacert /usr/share/elasticsearch/config/certs/ca/ca.crt -s https://localhost:5601 >/dev/null; if [[ $$? == 52 ]]; then echo 0; else echo 1; fi
        interval: 30s
        timeout: 10s
        retries: 5

    networks:
    docker_elk:


    secrets:
    ca.crt:
        file: ./config/ssl/ca/ca.crt
    logstash.keystore:
        file: ./config/logstash/logstash.keystore
    logstash.conf:
        file: ./config/logstash/pipeline/logstash.conf
    logstash.yml:
        file: ./config/logstash/logstash.yml
    elasticsearch.keystore:
        file: ./config/elasticsearch/elasticsearch.keystore
    elasticsearch.key:
        file: ./config/elasticsearch/elasticsearch.key
    elasticsearch.crt:
        file: ./config/elasticsearch/elasticsearch.crt
    elasticsearch.p12:
        file: ./config/elasticsearch/elasticsearch.p12
    es02.keystore:
        file: ./config/es02/es02.keystore
    es02.key:
        file: ./config/es02/es02.key
    es02.crt:
        file: ./config/es02/es02.crt
    es03.keystore:
        file: ./config/es03/es03.keystore
    es03.key:
        file: ./config/es03/es03.key
    es03.crt:
        file: ./config/es03/es03.crt
    kibana.yml:
        file: ./config/kibana/kibana.yml
    kibana.keystore:
        file: ./config/kibana/kibana.keystore
    kibana.key:
        file: ./config/kibana/kibana.key
    kibana.crt:
        file: ./config/kibana/kibana.crt

ˋˋˋ

##### Run it: docker-compose -f setup.yml up

![Imgur](https://i.imgur.com/73j0tPr.png)



# Conclusion

I am still trying to find out all the settings that can really affect the elasticsearch, but some of the setting are still testing. You are welcome to give any idea or setting that is better suit for ELK certificate, and this is running example of ELK multi node Certificate example. Give a try!!



## 來源

1. [**Elastic**](https://www.elastic.co/what-is/elk-stack)
2. [**docker-elk-tutorial**](https://github.com/twtrubiks/docker-elk-tutorial)
3. [**Learning ELK Stack**](https://www.amazon.com/Learning-ELK-Stack-Saurabh-Chhajed/dp/1785887157)
4. [**elk architecture**](https://www.atechref.com/blog/elk/elk-stack-architecture/)
5. [elastic/stack-docker](https://github.com/elastic/stack-docker)
6. [configuring-tls-docker](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/configuring-tls-docker.html)


