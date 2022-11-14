# Monitor Red Hat AMQ Streams

<!-- TOC -->

- [Monitor Red Hat AMQ Streams](#monitor-red-hat-amq-streams)
  - [Prerequisite](#prerequisite)
  - [JMX Exporter](#jmx-exporter)

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

* verify jmx libraries in /libs path such as jmx_prometheus_javaagent-xxxx.redhat-xxx.jar
  ```bash
  cd ~/amq-streams-2022/4-management/kafka/libs
  ll jmx*
  ```
  example result
  ```bash
  -rw-rw-r--. 1 student student 469023 Nov 14 05:04 jmx_prometheus_javaagent-0.16.1.redhat-00001.jar
  ```

* create zookeeper prometheus config file such as [zookeeper.yml](kafka/config/zookeeper.yml) and save file in /kafka/config of zookeeper server node.
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
* for zookeeper, edit /bin/zookeeper-server-start.sh by add config to load jmxagent jar file and set configuration to file zookeeper.yml in previous step such as
  ```bash
  #add below command for start jmx exporter at port 7075
  #set path to zookeeper.yml
  export EXTRA_ARGS=$EXTRA_ARGS' -javaagent:/home/student/amq-streams-2022/4-management/kafka/libs/jmx_prometheus_javaagent-0.16.1.redhat-00001.jar=7075:/home/student/amq-streams-2022/4-management/kafka/config/zookeeper.yml'

  exec $base_dir/kafka-run-class.sh $EXTRA_ARGS org.apache.zookeeper.server.quorum.QuorumPeerMain "$@"
  ```

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