# Access Control List

<!-- TOC -->

- [Access Control List](#access-control-list)
  - [Prerequisite](#prerequisite)
  - [Enabling authorization](#enabling-authorization)
  - [Adding ACL rules](#adding-acl-rules)

<!-- /TOC -->

## Prerequisite

* [Setup Red Hat AMQ Streams Lab](./../setup.md)
* Stop all server in previous lab
  * type ctrl+c in each terminal (stop kafka before stop zookeeper)
  * check kafka broker and zookeeper process with jps command
    ```bash
    jps
    ```
    
* clear old data in previous lab
  ```bash
  rm -rf /tmp/zookeep*
  rm -rf /tmp/kaf*
  ```

## Enabling authorization
* review [server.properties](./kafka/config/server.properties) in folder 5-basic-acl/kafka/bin/config/server.properties
* add authorizer.class.name to kafka.security.auth.SimpleAclAuthorizer in server.properties to enable simple authorization
  ```properties
  ...
  #Enabling authorization
  authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
  ```
* start zookeeper, open new terminal and run below command to start zookeeper
  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  ./kafka/bin/zookeeper-server-start.sh ./kafka/config/zookeeper.properties
  ```
* start kafka, open new terminal and run below command to start kafka
  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  ./kafka/bin/kafka-server-start.sh ./kafka/config/server.properties
  ```
* create topic "my-topic"
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my-topic --partitions 3 --replication-factor 1
  ```
* List the topics.
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```
## Adding ACL rules