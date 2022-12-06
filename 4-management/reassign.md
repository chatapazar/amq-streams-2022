# Reassign Partition

<!-- TOC -->

- [Reassign Partition](#reassign-partition)
  - [Prerequisite](#prerequisite)
  - [Start Cluster](#start-cluster)
    - [Topic Reassignments](#topic-reassignments)

<!-- /TOC -->

## Prerequisite

* [Setup Red Hat AMQ Streams Lab](./../setup.md)
* [Red Hat AMQ Streams Architecture](../2-amq-streams-architecture/architecture.md)

## Start Cluster

* make sure to stop all kafka & zookeeper server
  * by ctrl+c in your terminal or
  * call kafka-server-stop.sh for kafka broker and call zookeeper-server-stop.sh for zookeeper
  * make sure with jps command
* clear zookeeper & kafka data
  ```bash
  rm -rf /tmp/zookeeper*
  rm -rf /tmp/kafka*
  ```
  
* start the Zookeeper and Kafka cluster from the [Red Hat AMQ Streams Architecture](../2-amq-streams-architecture/architecture.md). This cluster will be used for this lab. run cli in each terminal (1 shell script 1 terminal)
  ```bash
  cd ~/amq-streams-2022/2-amq-streams-architecture/
  ./scripts/zookeeper-0.sh
  ./scripts/zookeeper-1.sh
  ./scripts/zookeeper-2.sh
  ./scripts/kafka-0.sh
  ./scripts/kafka-1.sh
  ./scripts/kafka-2.sh
  ```
  

### Topic Reassignments

* First, create a new topic which will be later used for reassigning:

  ```bash
  cd ~/amq-streams-2022/4-management/
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic reassignment-topic --partitions 1 --replication-factor 1
  ```
* view topic detial, keep information about topic
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic reassignment-topic
  ```
  example result, keep information about leader such as leader:0
  ```bash
  Topic: reassignment-topic       TopicId: uVag4sh4Sz-D7ifLi6OJFg PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=104857600
        Topic: reassignment-topic       Partition: 0    Leader: 0       Replicas: 0     Isr: 0
  ```
* load data to topic
  ```bash
  cd ~/amq-streams-2022/4-management/
  ./kafka/bin/kafka-producer-perf-test.sh \
  --topic reassignment-topic \
  --throughput -1 \
  --num-records 500000 \
  --record-size 2048 \
  --producer-props acks=all bootstrap.servers=localhost:9092,localhost:9093,localhost:9094
  ```
  
* Next, we generate reassignment file.
* Check the [topics-to-move.json](topics-to-move.json) file which tells the tool for which topics should the file be generated.
  * example of [topics-to-move.json](topics-to-move.json)
    ```json
    {
      "version": 1,
      "topics": [
        { "topic": "reassignment-topic"}
      ]
    }
    ```
  * generate reassign file, change broker-list to another value except value from leader info from previous describe topic command  (such as previous command leader:0, set broker-list to 1,2 from 0,1,2 or if leader:1, set broker-list to 0,2 etc.)
    ```bash
    cd ~/amq-streams-2022/4-management/
    ./kafka/bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --topics-to-move-json-file topics-to-move.json \
    --broker-list 1,2 \
    --generate
    ```
    example result 
    ```bash
    Current partition replica assignment
    {"version":1,"partitions":[{"topic":"reassignment-topic","partition":0,"replicas":[0],"log_dirs":["any"]}]}

    Proposed partition reassignment configuration
    {"version":1,"partitions":[{"topic":"reassignment-topic","partition":0,"replicas":[1],"log_dirs":["any"]}]}
    ```
  * copy this proposed partition reassignment and paste it in a local  [reassign-partition.json](reassign-partition.json) file
    ```json
    {"version":1,"partitions":[{"topic":"reassignment-topic","partition":0,"replicas":[1],"log_dirs":["any"]}]}
    ```
  
* Next we can use the prepared file to trigger the reassignment.

  ```bash
  cd ~/amq-streams-2022/4-management/
  export KAFKA_OPTS="-Djava.security.auth.login.config=../2-amq-streams-architecture/configs/kafka/jaas.config"
  ./kafka/bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassign-partition.json \
  --execute
  ```
  example result
  ```bash
  Current partition replica assignment

  {"version":1,"partitions":[{"topic":"reassignment-topic","partition":0,"replicas":[0],"log_dirs":["any"]}]}

  Save this to use as the --reassignment-json-file option during rollback
  Successfully started partition reassignment for reassignment-topic-0
  ```
* check reassign partition status 
  ```bash
  ./kafka/bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassign-partition.json \
  --verify
  ```
  example result
  ```bash
  Status of partition reassignment:
  Reassignment of partition reassignment-topic-0 is complete.

  Clearing broker-level throttles on brokers 0,1,2
  Clearing topic-level throttles on topic reassignment-topic
  ```
* check partition information again
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic reassignment-topic
  ```
  example result, make sure leader change, replicas change
  ```bash
  Topic: reassignment-topic       TopicId: uVag4sh4Sz-D7ifLi6OJFg PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=104857600
        Topic: reassignment-topic       Partition: 0    Leader: 1       Replicas: 1     Isr: 1
  ```

