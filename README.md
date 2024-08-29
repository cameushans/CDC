---
description: >-
  This document provide a overview of the implementation of change data capture
  using a mysql database as an upstream system and Maxwell daemon as a cdc tool
---

# Implementing CDC

First setup on the system hosting mysql DB



* [ ] Edit the /etc/my.cnf  according to the maxweel's needs

```bash
// Some code
# /etc/my.cnf

[mysqld]
# maxwell needs binlog_format=row
binlog_format=row
server_id=1 
log-bin=master
```

* [ ] Create a mysql user  for maxwell

```bash
// Some code
mysql> CREATE USER 'maxwell'@'%' IDENTIFIED BY 'XXXXXX';
mysql> CREATE USER 'maxwell'@'localhost' IDENTIFIED BY 'XXXXXX';

mysql> GRANT ALL ON maxwell.* TO 'maxwell'@'%';
mysql> GRANT ALL ON maxwell.* TO 'maxwell'@'localhost';
```

* [ ] Configure Maxwell for a replication setup in MySQL.

```bash
// Some code
mysql> GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
mysql> GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'localhost';

```

## Setup on the system hosting Maxwel program

* [x] Launch Maxwell with it's mysqls credentials&#x20;

```bash
// Some code
docker run -it --rm zendesk/maxwell bin/maxwell --user=$MYSQL_USERNAME \
    --password=$MYSQL_PASSWORD --host=$MYSQL_HOST --producer=stdout
```



* [x] Bootsraap of the kafka cluster using docker compose&#x20;

```bash
// docker compose for kafka cluster
version: '3.7'

services:
  zookeeper-1:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_SERVERS: zookeeper-1:2888:3888;zookeeper-2:2888:3888;zookeeper-3:2888:3888

  zookeeper-2:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2182:2182"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2182
      ZOOKEEPER_SERVER_ID: 2
      ZOOKEEPER_SERVERS: zookeeper-1:2888:3888;zookeeper-2:2888:3888;zookeeper-3:2888:3888
  
  zookeeper-3:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2183:2183"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2183
      ZOOKEEPER_SERVER_ID: 3
      ZOOKEEPER_SERVERS: zookeeper-1:2888:3888;zoozookeeper-2:2888:3888;zookeeper-3:2888:3888

  kafka-1:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka-1:19092,EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9092,DOCKER://host.docker.internal:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,DOCKER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper-1:2181,zookeeper-2:2182,zookeeper-3:2183"
      KAFKA_BROKER_ID: 1
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3

  kafka-2:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9093:9093"
      - "29093:29093"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka-2:19093,EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9093,DOCKER://host.docker.internal:29093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,DOCKER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper-1:2181,zookeeper-2:2182,zookeeper-3:2183"
      KAFKA_BROKER_ID: 2
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3

  kafka-3:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9094:9094"
      - "29094:29094"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka-3:19094,EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9094,DOCKER://host.docker.internal:29094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,DOCKER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper-1:2181,zookeeper-2:2182,zookeeper-3:2183"
      KAFKA_BROKER_ID: 3
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
```

* [x] After boostrapping your kafka cluster&#x20;

```bash
// Connect Maxwell with the kafka cluster
docker run -it --rm zendesk/maxwell bin/maxwell --user=$MYSQL_USERNAME \
    --password=$MYSQL_PASSWORD --host=$MYSQL_HOST --producer=kafka \
    --kafka.bootstrap.servers=$KAFKA_HOST:$KAFKA_PORT --kafka_topic=maxwell
```



```markup
// Architecture of the whole system

  +---------------------+
  |     MySQL Database  |
  |    (Source of Data) |
  +---------+-----------+
            |
            |
            v
  +---------+-----------+
  | Maxwell's Daemon    |
  | (CDC Tool)          |
  +---------+-----------+
            |
            |  Publishes
            |  Change Events
            v
  +---------+-----------+
  |       Kafka         |
  |  (Message Broker)   |
  +---------+-----------+
            |
            |  Topics
            v
  +---------+-----------+
  | Downstream Systems  |
  | (Consumers)         |
  +---------------------+

```
