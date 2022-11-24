# Cruise Control

<!-- TOC -->

- [Cruise Control](#cruise-control)
  - [Prerequisite](#prerequisite)
  - [Deploying the Cruise Control Metrics Reporter](#deploying-the-cruise-control-metrics-reporter)
  - [Prepare Test Topic](#prepare-test-topic)
  - [Enable Cruise Control](#enable-cruise-control)
  - [Test Auto Rebalance After Add Broker With Cruise-control](#test-auto-rebalance-after-add-broker-with-cruise-control)

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
  * check zookeeper & kafka process (3 zookeeper & 2 kafka)
    ```bash
    [student@node1 ~]$ jps
    1504 QuorumPeerMain
    2034 QuorumPeerMain
    4362 Jps
    3819 Kafka
    2557 QuorumPeerMain
    3277 Kafka
    ```
    or
    ```bash
    [student@node1 ~]$ jcmd
    1504 org.apache.zookeeper.server.quorum.QuorumPeerMain /home/student/amq-streams-2022/2-amq-streams-architecture/scripts/../configs/zookeeper/zookeeper-1.properties
    2034 org.apache.zookeeper.server.quorum.QuorumPeerMain /home/student/amq-streams-2022/2-amq-streams-architecture/scripts/../configs/zookeeper/zookeeper-2.properties
    3819 kafka.Kafka /home/student/amq-streams-2022/2-amq-streams-architecture/scripts/../configs/kafka/server-1.properties
    4380 jdk.jcmd/sun.tools.jcmd.JCmd
    2557 org.apache.zookeeper.server.quorum.QuorumPeerMain /home/student/amq-streams-2022/2-amq-streams-architecture/scripts/../configs/zookeeper/zookeeper-3.properties
    3277 kafka.Kafka /home/student/amq-streams-2022/2-amq-streams-architecture/scripts/../configs/kafka/server-0.properties
    ```
    
## Prepare Test Topic
* Create topic "demo"

  ```bash
  cd ~/amq-streams-2022/2-amq-streams-architecture/
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic demo --partitions 10 --replication-factor 2
  ```
  example result
  ```bash
  Created topic demo.
  ```
* Note: replication factor can't larger than available broker.
* Check Topic Description

  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
  ```
  example result, show partition of demo topic in broker 0 & broker 1
  ```bash
  Topic: demo     TopicId: gBQpzFJnRyGcvtFniabrVw PartitionCount: 10      ReplicationFactor: 2    Configs: segment.bytes=1073741824
        Topic: demo     Partition: 0    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: demo     Partition: 1    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: demo     Partition: 2    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: demo     Partition: 3    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: demo     Partition: 4    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: demo     Partition: 5    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: demo     Partition: 6    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: demo     Partition: 7    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: demo     Partition: 8    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: demo     Partition: 9    Leader: 0       Replicas: 0,1   Isr: 0,1
  ```
* load data to topic "demo"
  ./kafka/bin/kafka-producer-perf-test.sh \
  --topic demo \
  --throughput -1 \
  --num-records 500000 \
  --record-size 2048 \
  --producer-props acks=all bootstrap.servers=localhost:9092,localhost:9093

* run start kafka broker 2 in different terminal
    ```bash
    cd ~/amq-streams-2022/2-amq-streams-architecture/
    ./scripts/kafka-2.sh
    ```
* check all zookeeper & kafka process (3 zookeeper & 3 kafka)
    ```bash
    [student@node1 ~]$ jps
    1504 QuorumPeerMain
    6225 Kafka
    2034 QuorumPeerMain
    6760 Jps
    3819 Kafka
    2557 QuorumPeerMain
    3277 Kafka
    ```
* recheck topic information of "demo" don't change after start broker 2
  ```bash
  cd ~/amq-streams-2022/2-amq-streams-architecture/
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
  ```  

## Enable Cruise Control
* reviews cruise-control configuration in [cruisecontrol.properties](cruise-control/config/cruisecontrol.properties), Configure the properties used by Cruise Control and then start the Cruise Control server using the kafka-cruise-control-start.sh script. 
  ```properties
  ...
  # The Kafka cluster to control.
  bootstrap.servers=localhost:9092
  # The replication factor of Kafka metric sample store topic
  sample.store.topic.replication.factor=2

  # The configuration for the BrokerCapacityConfigFileResolver (supports JBOD, non-JBOD, and heterogeneous CPU core capacities)
  #capacity.config.file=config/capacity.json
  #capacity.config.file=config/capacityCores.json
  capacity.config.file=config/capacityJBOD.json

  # The list of goals to optimize the Kafka cluster for with pre-computed proposals
  default.goals=com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkInboundCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkOutboundCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.CpuCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.PotentialNwOutGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkInboundUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkOutboundUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.CpuUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.TopicReplicaDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.LeaderReplicaDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.LeaderBytesInDistributionGoal

  # The list of supported goals
  goals=com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkInboundCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkOutboundCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.CpuCapacityGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.PotentialNwOutGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkInboundUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkOutboundUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.CpuUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.TopicReplicaDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.LeaderReplicaDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.LeaderBytesInDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.kafkaassigner.KafkaAssignerDiskUsageDistributionGoal,com.linkedin.kafka.cruisecontrol.analyzer.kafkaassigner.KafkaAssignerEvenRackAwareGoal,com.linkedin.kafka.cruisecontrol.analyzer.goals.PreferredLeaderElectionGoal

  # How often should the cached proposal be expired and recalculated if necessary
  proposal.expiration.ms=60000

  # The zookeeper connect of the Kafka cluster
  zookeeper.connect=localhost:2181
  ```
* start cruise-contral at port 9191
  ```bash
  cd ~/amq-streams-2022/4-management/cruise-control
  ./kafka-cruise-control-start.sh config/cruisecontrol.properties 9191
  ```
* open new terminal, check cruise-control state
  ```bash
  curl 'http://localhost:9191/kafkacruisecontrol/state'
  ```
  example result
  ```bash
  [student@node1 ~]$ curl 'http://localhost:9191/kafkacruisecontrol/state'
  MonitorState: {state: RUNNING(0.000% trained), NumValidWindows: (0/0) (NaN%), NumValidPartitions: 0/0 (0.000%), flawedPartitions: 0}
  ExecutorState: {state: NO_TASK_IN_PROGRESS}
  AnalyzerState: {isProposalReady: false, readyGoals: []}
  AnomalyDetectorState: {selfHealingEnabled:[], selfHealingDisabled:[DISK_FAILURE, BROKER_FAILURE, GOAL_VIOLATION, METRIC_ANOMALY, TOPIC_ANOMALY, MAINTENANCE_EVENT], selfHealingEnabledRatio:{DISK_FAILURE=0.0, BROKER_FAILURE=0.0, GOAL_VIOLATION=0.0, METRIC_ANOMALY=0.0, TOPIC_ANOMALY=0.0, MAINTENANCE_EVENT=0.0}, recentGoalViolations:[], recentBrokerFailures:[], recentMetricAnomalies:[], recentDiskFailures:[], recentTopicAnomalies:[], recentMaintenanceEvents:[], metrics:{meanTimeBetweenAnomalies:{GOAL_VIOLATION:0.00 milliseconds, BROKER_FAILURE:0.00 milliseconds, METRIC_ANOMALY:0.00 milliseconds, DISK_FAILURE:0.00 milliseconds, TOPIC_ANOMALY:0.00 milliseconds}, meanTimeToStartFix:0.00 milliseconds, numSelfHealingStarted:0, numSelfHealingFailedToStart:0, ongoingAnomalyDuration=0.00 milliseconds}, ongoingSelfHealingAnomaly:None, balancednessScore:100.000}

  ```
* check Auto-created topics of cruise-control
  ```bash
  cd ~/amq-streams-2022/2-amq-streams-architecture/
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list --exclude-internal
  ```
  example result, show new topic such as "__CruiseControlMetrics", "__KafkaCruiseControlPartitionMetricSamples" and "__KafkaCruiseControlModelTrainingSamples"
  ```bash
  __CruiseControlMetrics
  __KafkaCruiseControlModelTrainingSamples
  __KafkaCruiseControlPartitionMetricSamples
  demo
  ```
* review capacity limits for Kafka broker resources in [config/config/capacityJBOD.json](./cruise-control/config/capacityJBOD.json) 
config/capacityJBOD.json, To apply the same capacity limits to every broker monitored by Cruise Control, set capacity limits for broker ID -1. To set different capacity limits for individual brokers, specify each broker ID and its capacity configuration.
  ```json
  {
    "brokerCapacities":[
      {
        "brokerId": "-1",
        "capacity": {
          "DISK": "100000",
          "CPU": "100",
          "NW_IN": "10000",
          "NW_OUT": "10000"
        },
        "doc": "This is the default capacity. Capacity unit used for disk is in MB, cpu is in percentage, network throughput is in KB."
      },
      {
        "brokerId": "0",
        "capacity": {
          "DISK": "100000",
          "CPU": "100",
          "NW_IN": "10000",
          "NW_OUT": "10000"
        },
        "doc": "This overrides the capacity for broker 0."
      }
    ]
  }
  ```
## Test Auto Rebalance After Add Broker With Cruise-control
* Test Generating optimization proposals with dryrun.
curl -v -X POST 'localhost:9191/kafkacruisecontrol/add_broker?brokerid=2'
Review the optimization proposal contained in the response.

Approving an optimization proposal
curl -v -X POST 'localhost:9191/kafkacruisecontrol/add_broker?dryrun=false&brokerid=2'

Check the progress
curl 'localhost:9191/kafkacruisecontrol/user_tasks'