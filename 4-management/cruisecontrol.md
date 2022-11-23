# Cruise Control

<!-- TOC -->

- [Cruise Control](#cruise-control)
  - [Prerequisite](#prerequisite)
  - [Deploying the Cruise Control Metrics Reporter](#deploying-the-cruise-control-metrics-reporter)
  - [Prepare Test Topic](#prepare-test-topic)

<!-- /TOC -->

## Prerequisite

* [Setup Red Hat AMQ Streams Lab](./../setup.md)
* [Red Hat AMQ Streams Architecture](../2-amq-streams-architecture/architecture.md)
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


## Deploying the Cruise Control Metrics Reporter
* In the Kafka configuration file [server-0.properties](../2-amq-streams-architecture/configs/kafka/server-0.properties) configure the Cruise Control Metrics Reporter:
* Add the CruiseControlMetricsReporter class to the metric.reporters configuration option. Do not remove any existing Metrics Reporters. (add at bottom of properties file)
  ```properties
  ...
  # add metric reporters
  metric.reporters=com.linkedin.kafka.cruisecontrol.metricsreporter.CruiseControlMetricsReporter
  cruise.control.metrics.topic.auto.create=true
  cruise.control.metrics.topic.num.partitions=1
  cruise.control.metrics.topic.replication.factor=1
  ```
* repeat with server-1.properties, server-2.properties in ~/amq-streams-2022/2-amq-streams-architecture/configs/kafka
* start cluster
  * run start zookeeper command in different terminal (1 shell script 1 terminal)
    ```bash
    cd ~/amq-streams-2022/2-amq-streams-architecture/
    ./scripts/zookeeper-0.sh
    ./scripts/zookeeper-1.sh
    ./scripts/zookeeper-2.sh
    ```
  * run start kafka broker command in different terminal, start only broker 0 and broker 1 (we will start broker 2 later in this lab)
    ```bash
    cd ~/amq-streams-2022/2-amq-streams-architecture/
    ./scripts/kafka-0.sh
    ./scripts/kafka-1.sh
    ```

## Prepare Test Topic
