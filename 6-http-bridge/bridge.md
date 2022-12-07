# HTTP Bridge

<!-- TOC -->

- [HTTP Bridge](#http-bridge)
  - [Prerequisite](#prerequisite)
  - [Kafka (HTTP) Bridge](#kafka-http-bridge)
  - [Install Kafka Bridge](#install-kafka-bridge)
  - [Configuring Kafka Bridge properties](#configuring-kafka-bridge-properties)

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

* start zookeeper
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/zookeeper-server-start.sh ./kafka/config/zookeeper.properties
  ```

* start kafka broker
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka-server-start.sh ./kafka/config/server.properties
  ```

## Kafka (HTTP) Bridge
* The Kafka Bridge provides a RESTful interface that allows HTTP-based clients to interact with a Kafka cluster.  It offers the advantages of a web API connection to AMQ Streams, without the need for client applications to interpret the Kafka protocol.

  The API has two main resources — consumers and topics — that are exposed and made accessible through endpoints to interact with consumers and producers in your Kafka cluster. The resources relate only to the Kafka Bridge, not the consumers and producers connected directly to Kafka.

## Install Kafka Bridge
* Download [Kafka Bridge](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=jboss.amq.streams&productChanged=yes) from Red Hat Web Site and Extract to your path
* layout of kafka http bridge after extract from zip file
  ```bash
  kafka-bridge
    ├── LICENSE
    ├── README.md
    ├── bin    
    ├── config
    └── libs
  ```

## Configuring Kafka Bridge properties
* You configure the Kafka Bridge, as any other Kafka client, using appropriate prefixes for Kafka-related properties.

  - "kafka." for general configuration that applies to producers and consumers, such as server connection and security.
  - "kafka.consumer." for consumer-specific configuration passed only to the consumer.
  - "kafka.producer." for producer-specific configuration passed only to the producer.

* Configure HTTP-related properties to enable HTTP access to the Kafka cluster.
  
  example of [application.properties](kafka-bridge/config/application.properties)
  ```properties
  kafka.bootstrap.servers=localhost:9092
  bridge.id=my-bridge
  http.enabled=true
  http.host=0.0.0.0
  http.port=8080 
  http.cors.enabled=true 
  http.cors.allowedOrigins=* 
  http.cors.allowedMethods=GET,POST,PUT,DELETE,OPTIONS,PATCH 
  ```
* Run the Kafka Bridge script using the configuration properties as a parameter:
  ```bash
  cd ~/amq-streams-2022/6-http-bridge
  .kafka-bridge/bin/kafka_bridge_run.sh --config-file=<path>/configfile.properties
  ```
