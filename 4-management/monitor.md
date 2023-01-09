# Monitor Red Hat AMQ Streams

<!-- TOC -->

- [Monitor Red Hat AMQ Streams](#monitor-red-hat-amq-streams)
  - [Prerequisite](#prerequisite)
  - [JMX Exporter](#jmx-exporter)
    - [JMX Library](#jmx-library)
    - [JMX Zookeeper](#jmx-zookeeper)
    - [JMX Kafka](#jmx-kafka)
  - [Kafka Exporter](#kafka-exporter)
    - [Prepare Topic Data](#prepare-topic-data)
    - [Run Kafka Exporter](#run-kafka-exporter)

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
 
## JMX Exporter

### JMX Library
* verify jmx libraries in /libs path such as jmx_prometheus_javaagent-xxxx.redhat-xxx.jar
  ```bash
  cd ~/amq-streams-2022/4-management/kafka/libs
  ll jmx*
  ```
  example result
  ```bash
  -rw-rw-r--. 1 student student 469023 Nov 14 05:04 jmx_prometheus_javaagent-0.16.1.redhat-00001.jar
  ```
### JMX Zookeeper
* create zookeeper prometheus config file such as [zookeeper.yml](kafka/config/zookeeper.yml) and save file in ~/amq-streams-2022/4-management/kafka/config of zookeeper server node.
  ```yml
  lowercaseOutputName: true
  lowercaseOutputLabelNames: true
  ...
  rules:
  # Below rule applies for Zookeeper Cluster having multiple ZK nodes
  # org.apache.ZooKeeperService:name0=*,name3=Connections,name1=*,name2=*,name4=*,name5=*
  - pattern: "org.apache.ZooKeeperService<name0=(.+), name1=replica.(\\d+), name2=(\\w+), name3=Connections, name4=(.+), name5=(.+)><>([^:]+)"
  name: zookeeper_connections_$6
  labels:
    server_name: "$1"
    server_id: $2
    client_address: "$4"
    connection_id: "$5"
    member_type: "$3"
  ...
  ```
* for zookeeper, edit ~/amq-streams-2022/4-management/kafka/bin/zookeeper-server-start.sh by add config to load jmxagent jar file and set configuration to file zookeeper.yml in previous step such as
  ```bash
  #add below command for start jmx exporter at port 7075
  #set path to zookeeper.yml
  export EXTRA_ARGS=$EXTRA_ARGS' -javaagent:/home/student/amq-streams-2022/4-management/kafka/libs/jmx_prometheus_javaagent-0.16.1.redhat-00001.jar=7075:/home/student/amq-streams-2022/4-management/kafka/config/zookeeper.yml'

  exec $base_dir/kafka-run-class.sh $EXTRA_ARGS org.apache.zookeeper.server.quorum.QuorumPeerMain "$@"
  ```
* start zookeeper
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/zookeeper-server-start.sh ./kafka/config/zookeeper.properties
  ```
* check port 7075 start with command, open new terminal
  ```bash
  netstat -tnlp
  ```
  example result
  ```bash
  ...
  tcp6       0      0 :::7075                 :::*                    LISTEN      16617/java
  ...
  ```
* call curl to http://localhost:7075/metrics for check metrics work!
  ```bash
  curl http://localhost:7075/metrics
  ```
  example result
  ```bash
  ...
  # TYPE jmx_exporter_build_info gauge
  jmx_exporter_build_info{version="0.16.1.redhat-00001",name="jmx_prometheus_javaagent",} 1.0
  # HELP jmx_config_reload_failure_created Number of times configuration have failed to be reloaded.
  # TYPE jmx_config_reload_failure_created gauge
  jmx_config_reload_failure_created 1.668411562813E9
  # HELP jmx_config_reload_success_created Number of times configuration have successfully been reloaded.
  # TYPE jmx_config_reload_success_created gauge
  jmx_config_reload_success_created 1.668411562812E9
  # HELP jvm_memory_pool_allocated_bytes_created Total bytes allocated in a given JVM memory pool. Only updated after GC, not continuously.
  # TYPE jvm_memory_pool_allocated_bytes_created gauge
  jvm_memory_pool_allocated_bytes_created{pool="CodeHeap 'profiled nmethods'",} 1.668411563609E9
  jvm_memory_pool_allocated_bytes_created{pool="G1 Old Gen",} 1.668411563616E9
  jvm_memory_pool_allocated_bytes_created{pool="G1 Eden Space",} 1.668411563616E9
  jvm_memory_pool_allocated_bytes_created{pool="CodeHeap 'non-profiled nmethods'",} 1.668411563616E9
  jvm_memory_pool_allocated_bytes_created{pool="G1 Survivor Space",} 1.668411563616E9
  jvm_memory_pool_allocated_bytes_created{pool="Compressed Class Space",} 1.668411563616E9
  jvm_memory_pool_allocated_bytes_created{pool="Metaspace",} 1.668411563616E9
  jvm_memory_pool_allocated_bytes_created{pool="CodeHeap 'non-nmethods'",} 1.668411563616E9
  ```
  
### JMX Kafka
* create kafka broker prometheus config file such as [kafka_broker.yml](kafka/config/kafka_broker.yml) and save file in /kafka/config of kafka broker node.
  ```yml
  ...
  #startDelaySeconds: 120
  lowercaseOutputName: true
  lowercaseOutputLabelNames: true
  ...
  rules:
  # This is by far the biggest contributor to the number of sheer metrics being produced.
  # Always keep it on the top for the case of probability when so many metrics will hit the first condition and exit.
  # "kafka.cluster:type=*, name=*, topic=*, partition=*"
  # "kafka.log:type=*,name=*, topic=*, partition=*"
  - pattern: kafka.(\w+)<type=(.+), name=(.+), topic=(.+), partition=(.+)><>Value
    name: kafka_$1_$2_$3
    type: GAUGE
    labels:
        topic: "$4"
        partition: "$5"
  ...
  ```
* for kafka broker, edit ~/amq-streams-2022/4-management/kafka/bin/kafka-server-start.sh by add config to load jmxagent jar file and set configuration to file kafka_broker.yml in previous step such as
  ```bash
  #add below command for start jmx exporter at port 7076
  #set path to kafka_broker.yml
  export KAFKA_OPTS=$KAFKA_OPTS' -javaagent:/home/student/amq-streams-2022/4-management/kafka/libs/jmx_prometheus_javaagent-0.16.1.redhat-00001.jar=7076:/home/student/amq-streams-2022/4-management/kafka/config/kafka_broker.yml'

  exec $base_dir/kafka-run-class.sh $EXTRA_ARGS kafka.Kafka "$@"
  ```
* start kafka broker
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka-server-start.sh ./kafka/config/server.properties
  ```
* check port 7076 start with command, open new terminal
  ```bash
  netstat -tnlp
  ```
  example result
  ```bash
  ...
  tcp6       0      0 :::7076                 :::*                    LISTEN      16617/java
  ...
  ```
* call curl to http://localhost:7076/metrics for check metrics work!
  ```bash
  curl http://localhost:7076/metrics
  ```
  example result
  ```bash
  ...
  # TYPE jmx_config_reload_success_created gauge
  jmx_config_reload_success_created 1.668413540288E9
  # HELP jvm_memory_pool_allocated_bytes_created Total bytes allocated in a given JVM memory pool. Only updated after GC, not continuously.
  # TYPE jvm_memory_pool_allocated_bytes_created gauge
  jvm_memory_pool_allocated_bytes_created{pool="CodeHeap 'profiled nmethods'",} 1.668413541537E9
  jvm_memory_pool_allocated_bytes_created{pool="G1 Old Gen",} 1.66841354154E9
  jvm_memory_pool_allocated_bytes_created{pool="G1 Eden Space",} 1.66841354154E9
  jvm_memory_pool_allocated_bytes_created{pool="CodeHeap 'non-profiled nmethods'",} 1.66841354154E9
  jvm_memory_pool_allocated_bytes_created{pool="G1 Survivor Space",} 1.66841354154E9
  jvm_memory_pool_allocated_bytes_created{pool="Compressed Class Space",} 1.66841354154E9
  jvm_memory_pool_allocated_bytes_created{pool="Metaspace",} 1.66841354154E9
  jvm_memory_pool_allocated_bytes_created{pool="CodeHeap 'non-nmethods'",} 1.66841354154E9
  ```

## Kafka Exporter

### Prepare Topic Data
* List the topics using Kafka, open new terminal and run command
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```
* no topic show in terminal
* create sample new topic which we will use
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my-topic --partitions 3 --replication-factor 1
  ```
  result of create topic command
  ```bash
  Created topic my-topic.
  ```
* List the topics again to see it was created.
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```
* Describe the topic to see more details:
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic my-topic --describe
  ```
* start consumer, open new terminal and run command
  ```bash
  ./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-topic --from-beginning --group monitor
  ```
* Start the console producer for create and send message to topic
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-topic
  ```
* Wait until it is ready (it should show `>`). and send 3 messages.
  ```bash
  >a
  >b
  >c
  >
* stop consumer console by type ctrl+c
* back to producer console and submit 3 more data
  ```bash
  >a
  >b
  >c
  >d
  >e
  >f
  >
  ``` 
* stop producer console 
* List the topics again to see '__consumer_offsets' was created.
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```
  example result
  ```bash
  __consumer_offsets
  my-topic
  ```
* monitor consumer groups:
  ```bash
  ./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --list
  ```
* and describe them: see current-offset, last-offset, lag

  ```bash
  ./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --describe
  ```
  example result
  ```bash
  Consumer group 'monitor' has no active members.

  GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
  monitor         my-topic        0          1               3               2               -               -               -
  monitor         my-topic        1          1               1               0               -               -               -
  monitor         my-topic        2          1               2               1               -               -               -
  ```

### Run Kafka Exporter
* run kafka exporter
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka_exporter --kafka.version=3.2.3 --kafka.server=localhost:9092
  ```
  example result
  ```bash
  I1114 09:30:43.593105   74303 kafka_exporter.go:792] Starting kafka_exporter (version=1.6.0, branch=HEAD, revision=c021e94dfb808e642d41064c6550cbba87fe30c6)
  I1114 09:30:43.601413   74303 kafka_exporter.go:963] Listening on HTTP :9308
  ```
* call curl to http://localhost:9308/metrics for check metrics work!
  ```bash
  curl http://localhost:9308/metrics
  ```
  
  example result  
  ```bash
  ...
  # TYPE kafka_consumergroup_lag_sum gauge
  kafka_consumergroup_lag_sum{consumergroup="monitor",topic="my-topic"} 3
  ...
  # TYPE kafka_topic_partition_current_offset gauge
  kafka_topic_partition_current_offset{partition="0",topic="__consumer_offsets"} 0
  kafka_topic_partition_current_offset{partition="0",topic="my-topic"} 3
  kafka_topic_partition_current_offset{partition="1",topic="__consumer_offsets"} 0
  kafka_topic_partition_current_offset{partition="1",topic="my-topic"} 1
  ...
  kafka_topic_partition_current_offset{partition="2",topic="my-topic"} 2
  ...
  kafka_topic_partition_oldest_offset{partition="0",topic="__consumer_offsets"} 0
  kafka_topic_partition_oldest_offset{partition="0",topic="my-topic"} 0
  kafka_topic_partition_oldest_offset{partition="1",topic="__consumer_offsets"} 0
  kafka_topic_partition_oldest_offset{partition="1",topic="my-topic"} 0
  ...
  kafka_topic_partitions{topic="my-topic"} 3
  ...
  ```