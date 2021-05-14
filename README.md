# Kafka-ELK-Pipeline
Setup LogData Pipeline using FileBeat->kafka->logstash->ELK in 15 Minutes (POC)



Requirements
============
	• Unix host with at least 3GB RAM with 20 GB of space( Here I am using Centos-7).
	• Docker and Docker-compose installed.
	• Working Internet connection. 

Create directory for Pipeline.
```
mkdir data-pipeline
cd data-pipeline
touch Install-Stack.yaml

```
Install-Stack.yaml  -- > Docker Compose file

############### KAFKA Component ###############
```
version: "3"
services:
  zookeeper:
    image: zookeeper
    restart: always
    container_name: zookeeper
    hostname: zookeeper
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
  kafka:
    image: wurstmeister/kafka
    container_name: kafka
    ports:
    - 9092:9092
    - 8004:8004
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 192.168.31.12
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=kafka -Dcom.sun.management.jmxremote.rmi.port=8004"
      JMX_PORT: 8004
  kafka_manager:
    image: hlebalbau/kafka-manager:stable
    container_name: kakfa-manager
    restart: always
    ports:
      - "9000:9000"
    environment:
      ZK_HOSTS: "zookeeper:2181"
      APPLICATION_SECRET: "random-secret"
    command: -Dpidfile.path=/dev/null

####### Elastic Search Component ##########
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.4.0
    container_name: elasticsearch
    restart: always
    environment:
      - xpack.security.enabled=false
      - discovery.type=single-node
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    cap_add:
      - IPC_LOCK
    volumes:
      - elasticsearch-data-volume:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
  kibana:
    container_name: kibana
    image: docker.elastic.co/kibana/kibana:7.4.0
    restart: always
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200    # address of elasticsearch docker container which kibana will connect
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch                                   # kibana will start when elasticsearch has started
volumes:
  elasticsearch-data-volume:
    driver: local
```
===================================================================================================
Open the required port of firewall
```
sudo firewall-cmd --permanent --add-port=9092/tcp
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --add-port=5601/tcp
sudo firewall-cmd --permanent --add-port=2181/tcp
sudo firewall-cmd --permanent --add-port=3888/tcp
sudo firewall-cmd --permanent --add-port=2888/tcp
sudo firewall-cmd --permanent --add-port=8004/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```


Docker commands
===============
 To start
```
# docker-compose -f install-stack.yaml up -d
``` 
 To stop
```
# docker-compose -f install-stack.yaml down
```
To get The Status
```
# docker ps
```
====== Docker Output ======
```
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS                                                  NAMES
34ae842aa9eb        docker.elastic.co/kibana/kibana:7.4.0                 "/usr/local/bin/du..."   18 minutes ago      Up 18 minutes       0.0.0.0:5601->5601/tcp                                 kibana
6ced2e7bd044        wurstmeister/kafka                                    "start-kafka.sh"         18 minutes ago      Up 18 minutes       0.0.0.0:9092->9092/tcp                                 kafka
ea4a13d84c1a        docker.elastic.co/elasticsearch/elasticsearch:7.4.0   "/usr/local/bin/do..."   18 minutes ago      Up 18 minutes       0.0.0.0:9200->9200/tcp, 9300/tcp                       elasticsearch
afdece028175        zookeeper                                             "/docker-entrypoin..."   18 minutes ago      Up 18 minutes       2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, 8080/tcp   zookeeper
73399d09fc8c        hlebalbau/kafka-manager:stable                        "/kafka-manager/bi..."   18 minutes ago      Up 18 minutes       0.0.0.0:9000->9000/tcp                                 kakfa-manager
```
====================================
Installing and Configuring Filebeat:
====================================
```
sudo curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.9.3-linux-x86_64.tar.gz 
sudo tar xzvf filebeat-7.9.3-linux-x86_64.tar.gz
cd filebeat-7.9.3-linux-x86_64
sudo ./filebeat modules list
sudo ./filebeat modules enable system
Then Use below link to update:  filebeat.yml
cp -p filebeat.yml filebeat.yml.bak
https://www.elastic.co/guide/en/logstash/current/use-filebeat-modules-kafka.html

sudo vi filebeat.yml
# ================================== Outputs ===================================

# Configure what destination to use when sending the data collected by the beat.

# ---------------------------- Elasticsearch Output ----------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
#  hosts: ["localhost:9200"]
output.kafka:
  hosts: ["192.168.31.12:9092"]
  topics:
    - topic: "Elasticsearch"
    - topic: "loki"
  codec.json:
    pretty: false
```
========================================================
Installing and configuring logstash
========================================================
https://www.elastic.co/guide/en/logstash/current/installing-logstash.html     -- to install

Post installation
=================
```
#sudo systemctl stop logstash

#cd /etc/logstash/conf.d
#sudo vi pipeline.conf  -- Add below config
======
input {
    kafka {
            bootstrap_servers => "192.168.31.12:9092"
            topics => ["Elasticsearch"]
    }
}

output {
   elasticsearch {
      hosts => ["192.168.31.12:9200"]
      index => "elastic-search"
      workers => 1
    }
}

```
======
Start The Logstash
```
#sudo systemctl start logstash
```
=======

Creating the Topic : Elasticsearch on Kafka.
	1. Login to kafka-manager 127.0.0.1:9000
![image](https://user-images.githubusercontent.com/31564143/118267968-6842a380-b4da-11eb-8a19-07cb05ca3e34.png)
