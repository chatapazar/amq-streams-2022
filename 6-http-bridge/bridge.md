# HTTP Bridge

<!-- TOC -->

- [HTTP Bridge](#http-bridge)
  - [Prerequisite](#prerequisite)
  - [Kafka (HTTP) Bridge](#kafka-http-bridge)
  - [Install Kafka Bridge](#install-kafka-bridge)
  - [Configuring Kafka Bridge properties](#configuring-kafka-bridge-properties)
  - [Producing messages to topics and partitions](#producing-messages-to-topics-and-partitions)
  - [Creating a Kafka Bridge consumer](#creating-a-kafka-bridge-consumer)
  - [Subscribing a Kafka Bridge consumer to topics](#subscribing-a-kafka-bridge-consumer-to-topics)
  - [Retrieving the latest messages from a Kafka Bridge consumer](#retrieving-the-latest-messages-from-a-kafka-bridge-consumer)

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
  ./kafka-bridge/bin/kafka_bridge_run.sh --config-file=/home/student/amq-streams-2022/6-http-bridge/kafka-bridge/config/application.properties
  ```
  
  example result
  ```bash
  [2022-12-07 07:12:20,685] INFO  <ppInfoParser:119> [oop-thread-1] Kafka version: 2.8.0.redhat-00009
  [2022-12-07 07:12:20,685] INFO  <ppInfoParser:120> [oop-thread-1] Kafka commitId: 7f53221f4d51179a
  [2022-12-07 07:12:20,685] INFO  <ppInfoParser:121> [oop-thread-1] Kafka startTimeMs: 1670397140682
  [2022-12-07 07:12:20,858] INFO  <HttpBridge  :104> [oop-thread-1] HTTP-Kafka Bridge started and listening on port 8080
  [2022-12-07 07:12:20,858] INFO  <HttpBridge  :105> [oop-thread-1] HTTP-Kafka Bridge bootstrap servers localhost:9092
  [2022-12-07 07:12:20,871] INFO  <Application :224> [oop-thread-0] HTTP verticle instance deployed [e3a5f04b-f6fa-4e67-8485-7b4b7506f80b]
  ```

## Producing messages to topics and partitions
* create "bridge-quickstart-topic" topic
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic bridge-quickstart-topic --partitions 3 --replication-factor 1
  ```
* Verifying the topic was created
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic bridge-quickstart-topic
  ```
* Using the Kafka Bridge, produce messages to the topic you created:
  ```bash
  curl -X POST \
    http://localhost:8080/topics/bridge-quickstart-topic \
    -H 'content-type: application/vnd.kafka.json.v2+json' \
    -d '{
      "records": [
        {
            "key": "my-key",
            "value": "sales-lead-0001"
        },
        {
            "value": "sales-lead-0002",
            "partition": 2
        },
        {
            "value": "sales-lead-0003"
        },
        {
            "value": "sales-lead-0004"
        },
        {
            "value": "sales-lead-0005"
        },
        {
            "value": "sales-lead-0006"
        },
        {
            "value": "sales-lead-0007"
        },
        {
            "value": "sales-lead-008"
        },
        {
            "value": "sales-lead-0009"
        },
        {
            "value": "sales-lead-0010"
        }
      ]
  }'
  ```
  
  If the request is successful, the Kafka Bridge returns an offsets array, along with a 200 code and a content-type header of application/vnd.kafka.v2+json. For each message, the offsets array describes:
  - The partition that the message was sent to
  - The current message offset of the partition
  
  Example response
  ```bash
  #...
    {
    "offsets":[
        {
        "partition":0,
        "offset":0
        },
        {
        "partition":2,
        "offset":0
        },
        {
        "partition":0,
        "offset":1
        }
    ]
    }
  ```

* List Topics
  ```bash
  curl -X GET http://localhost:8080/topics
  ```

  example result
  ```bash
  ["bridge-quickstart-topic"]
  ```

* Get topic configuration and partition details
  ```bash
  curl -X GET http://localhost:8080/topics/bridge-quickstart-topic
  ```

  example response
  ```bash
  {
    "name": "bridge-quickstart-topic",
    "configs": {
        "compression.type": "producer",
        "leader.replication.throttled.replicas": "",
        "min.insync.replicas": "1",
        "message.downconversion.enable": "true",
        "segment.jitter.ms": "0",
        "cleanup.policy": "delete",
        "flush.ms": "9223372036854775807",
        "follower.replication.throttled.replicas": "",
        "segment.bytes": "1073741824",
        "retention.ms": "604800000",
        "flush.messages": "9223372036854775807",
        "message.format.version": "2.8-IV1",
        "max.compaction.lag.ms": "9223372036854775807",
        "file.delete.delay.ms": "60000",
        "max.message.bytes": "1048588",
        "min.compaction.lag.ms": "0",
        "message.timestamp.type": "CreateTime",
        "preallocate": "false",
        "index.interval.bytes": "4096",
        "min.cleanable.dirty.ratio": "0.5",
        "unclean.leader.election.enable": "false",
        "retention.bytes": "-1",
        "delete.retention.ms": "86400000",
        "segment.ms": "604800000",
        "message.timestamp.difference.max.ms": "9223372036854775807",
        "segment.index.bytes": "10485760"
    },
    "partitions": [
        {
        "partition": 0,
        "leader": 0,
        "replicas": [
            {
            "broker": 0,
            "leader": true,
            "in_sync": true
            },
            {
            "broker": 1,
            "leader": false,
            "in_sync": true
            },
            {
            "broker": 2,
            "leader": false,
            "in_sync": true
            }
        ]
        },
        {
        "partition": 1,
        "leader": 2,
        "replicas": [
            {
            "broker": 2,
            "leader": true,
            "in_sync": true
            },
            {
            "broker": 0,
            "leader": false,
            "in_sync": true
            },
            {
            "broker": 1,
            "leader": false,
            "in_sync": true
            }
        ]
        },
        {
        "partition": 2,
        "leader": 1,
        "replicas": [
            {
            "broker": 1,
            "leader": true,
            "in_sync": true
            },
            {
            "broker": 2,
            "leader": false,
            "in_sync": true
            },
            {
            "broker": 0,
            "leader": false,
            "in_sync": true
            }
        ]
        }
    ]
    }
  ```
* try to test with another function
  * List the partitions of a specific topic
    ```bash
    curl -X GET http://localhost:8080/topics/bridge-quickstart-topic/partitions
    ```
  * List the details of a specific topic partition
    ```bash
    curl -X GET http://localhost:8080/topics/bridge-quickstart-topic/partitions/0
    ```
  * List the offsets of a specific topic partition
   ```bash
   curl -X GET http://localhost:8080/topics/bridge-quickstart-topic/partitions/0/offsets
   ```

## Creating a Kafka Bridge consumer
* Create a Kafka Bridge consumer in a new consumer group named bridge-quickstart-consumer-group:
  ```bash
  curl -X POST http://localhost:8080/consumers/bridge-quickstart-consumer-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "bridge-quickstart-consumer",
    "auto.offset.reset": "earliest",
    "format": "json",
    "enable.auto.commit": false,
    "fetch.min.bytes": 512,
    "consumer.request.timeout.ms": 30000
  }'
  ```
  - The consumer is named bridge-quickstart-consumer and the embedded data format is set as json.
  - Some basic configuration settings are defined.
  - The consumer will not commit offsets to the log automatically because the enable.auto.commit setting is false. You will commit the offsets manually later in this quickstart.

    If the request is successful, the Kafka Bridge returns the consumer ID (instance_id) and base URL (base_uri) in the response body, along with a 200 code.

  example response
  ```bash
   {"instance_id":"bridge-quickstart-consumer","base_uri":"http://localhost:8080/consumers/bridge-quickstart-consumer-group/instances/bridge-quickstart-consumer"}
  ```  
  - Copy the base URL (base_uri) to use in the other consumer operations in this quickstart.

## Subscribing a Kafka Bridge consumer to topics
* Subscribe the consumer to the bridge-quickstart-topic topic that you created earlier (If the request is successful, the Kafka Bridge returns a 204 (No Content) code only.):
  ```bash
  curl -X POST http://localhost:8080/consumers/bridge-quickstart-consumer-group/instances/bridge-quickstart-consumer/subscription \
    -H 'content-type: application/vnd.kafka.v2+json' \
    -d '{
        "topics": [
            "bridge-quickstart-topic"
        ]
    }'
  ```

## Retrieving the latest messages from a Kafka Bridge consumer
* Retrieve the latest messages from the Kafka Bridge consumer by requesting data from the records endpoint. In production, HTTP clients can call this endpoint repeatedly (in a loop).
* Submit a GET request to the records endpoint:
  ```bash
  curl -X GET http://localhost:8080/consumers/bridge-quickstart-consumer-group/instances/bridge-quickstart-consumer/records \
  -H 'accept: application/vnd.kafka.json.v2+json'
  ```
  After creating and subscribing to a Kafka Bridge consumer, a first GET request will return an empty response because the poll operation starts a rebalancing process to assign partitions.

* Repeat previous step to retrieve messages from the Kafka Bridge consumer. The Kafka Bridge returns an array of messages — describing the topic name, key, value, partition, and offset — in the response body, along with a 200 code. Messages are retrieved from the latest offset by default. (repeat call until have response from bridge)
  ```bash
  curl -X GET http://localhost:8080/consumers/bridge-quickstart-consumer-group/instances/bridge-quickstart-consumer/records \
  -H 'accept: application/vnd.kafka.json.v2+json'
  ```
  
  example result
  ```bash
  [student@node1 4-management]$ curl -X GET http://localhost:8080/consumers/bridge-quickstart-consumer-group/instances/bridge-quickstart-consumer/records   -H 'acpt: application/vnd.kafka.json.v2+json'
  []
  [student@node1 4-management]$ curl -X GET http://localhost:8080/consumers/bridge-quickstart-consumer-group/instances/bridge-quickstart-consumer/records   -H 'acpt: application/vnd.kafka.json.v2+json'
  [{"topic":"bridge-quickstart-topic","key":"my-key","value":"sales-lead-0001","partition":0,"offset":0},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0003","partition":0,"offset":1},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0004","partition":0,"offset":2},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0005","partition":0,"offset":3},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0006","partition":0,"offset":4},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0007","partition":0,"offset":5},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-008","partition":0,"offset":6},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0009","partition":0,"offset":7},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0010","partition":0,"offset":8},{"topic":"bridge-quickstart-topic","key":null,"value":"sales-lead-0002","partition":2,"offset":0}]
  [student@node1 4-management]$ curl -X GET http://localhost:8080/consumers/bridge-quickstart-consumer-group/instances/bridge-quickstart-consumer/records   -H 'acpt: application/vnd.kafka.json.v2+json'
  []
  ```