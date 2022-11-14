# Reassign Partition

<!-- TOC -->

- [Reassign Partition](#reassign-partition)
  - [Prerequisite](#prerequisite)
  - [Start Cluster](#start-cluster)
    - [Topics](#topics)
    - [Consumer groups](#consumer-groups)
    - [Topic Reassignments](#topic-reassignments)

<!-- /TOC -->

## Prerequisite

* [Setup Red Hat AMQ Streams Lab](./../setup.md)
* [Red Hat AMQ Streams Architecture](../2-amq-streams-architecture/architecture.md)

## Start Cluster

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
  
### Topics
* check demo topic already in cluster
  ```bash
  cd ~/amq-streams-2022/4-management/
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```
  example result
  ```bash
  __consumer_offsets
  demo
  ```
* We already used a lot of commands. You can also use the script to show only some topics in _troubles_:
  * '--under-replicated-partitions' --> displays the number of partitions that do not have enough replicas to meet the desired replication factor.
  * '--unavailable-partitions' --> list partitions that currently don't have a leader and hence cannot be used by Consumers or Producers.

  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --unavailable-partitions
  ```

### Consumer groups

* `kafka-consumer-groups.sh` lets you manage and monitor consumer groups:

  ```bash
  ./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --list
  ```

* or describe them:

  ```bash
  ./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --describe
  ```

* Reset the offset to 0:

  ```bash
  ./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --reset-offsets --to-earliest --group replay-group --topic demo --execute
  ```

* Reset the offset to last message:

  ```bash
  ./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --reset-offsets --to-latest --group replay-group --topic demo --execute
  ```

### Topic Reassignments

* First, create a new topic which will be later used for reassigning:

  ```bash
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
  * generate reassign file, change broker-list to another value except value from leader info from previous describe topic command  (such as previous command leader:0, set broker-list to 1,2 from 0,1,2)
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
  * copy this proposed partition reassignment and paste it in a local [reassign-partition.json](reassign-partition.json) file
    ```json
    {"version":1,"partitions":[{"topic":"reassignment-topic","partition":0,"replicas":[1],"log_dirs":["any"]}]}
    ```
  
* Next we can use the prepared file to trigger the reassignment.

  ```bash
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
  example result
  ```bash
  Topic: reassignment-topic       TopicId: uVag4sh4Sz-D7ifLi6OJFg PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=104857600
        Topic: reassignment-topic       Partition: 0    Leader: 1       Replicas: 1     Isr: 1
  ```

