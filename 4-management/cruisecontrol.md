# Cruise Control

<!-- TOC -->

- [Cruise Control](#cruise-control)
  - [Prerequisite](#prerequisite)
  - [Deploying the Cruise Control Metrics Reporter](#deploying-the-cruise-control-metrics-reporter)
  - [Prepare Test Topic](#prepare-test-topic)
  - [Enable Cruise Control](#enable-cruise-control)
  - [Test Auto Rebalance After Add Broker With Cruise-control](#test-auto-rebalance-after-add-broker-with-cruise-control)
  - [Stop Server](#stop-server)

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
  ```bash
  ./kafka/bin/kafka-producer-perf-test.sh \
  --topic demo \
  --throughput -1 \
  --num-records 500000 \
  --record-size 2048 \
  --producer-props acks=all bootstrap.servers=localhost:9092,localhost:9093
  ```

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
* review capacity limits for Kafka broker resources in [config/config/capacityJBOD.json](./cruise-control/config/capacityJBOD.json) 
, To apply the same capacity limits to every broker monitored by Cruise Control, set capacity limits for broker ID -1. To set different capacity limits for individual brokers, specify each broker ID and its capacity configuration.
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
  ```bash
  curl -v -X POST 'localhost:9191/kafkacruisecontrol/add_broker?brokerid=2'
  ```
  example result, Review the optimization proposal contained in the response.
  ```bash
  *   Trying ::1...
  * TCP_NODELAY set
  * Connected to localhost (::1) port 9191 (#0)
  > POST /kafkacruisecontrol/add_broker?brokerid=2 HTTP/1.1
  > Host: localhost:9191
  > User-Agent: curl/7.61.1
  > Accept: */*
  >
  < HTTP/1.1 200 OK
  < Date: Thu, 24 Nov 2022 06:17:17 GMT
  < Set-Cookie: JSESSIONID=node0vavr3i7abj7ilf5myhd9ixsb8.node0; Path=/
  < Expires: Thu, 01 Jan 1970 00:00:00 GMT
  < User-Task-ID: 01d01442-230e-4f39-9514-5355239f99e1
  < Content-Type: text/plain;charset=utf-8
  < Cruise-Control-Version: 2.5.89.redhat-00006
  < Cruise-Control-Commit_Id: 1e4e2dd8f14df6c409244c11c84c52038597ca99
  < Content-Length: 13200
  < Server: Jetty(9.4.45.v20220203-redhat-00001)
  <


  Optimization has 9 inter-broker replica(593 MB) moves, 0 intra-broker replica(0 MB) moves and 0 leadership moves with a cluster model of 1 recent windows and 100.000% of the partitions covered.
  Excluded Topics: [].
  Excluded Brokers For Leadership: [].
  Excluded Brokers For Replica Move: [].
  Counts: 3 brokers 149 replicas 4 topics.
  On-demand Balancedness Score Before (86.026) After(92.120).
  Provision Status: RIGHT_SIZED.

  [     2 ms] Stats for ReplicaCapacityGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     985.028 potentialNwOut:     556.011 replicas:53 leaderReplicas:27 topicReplicas:22}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:       0.000 potentialNwOut:       0.000 replicas:43 leaderReplicas:22 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:     464.343 potentialNwOut:     262.106 replicas:4.714045207910317 leaderReplicas:2.1602468994692865 topicReplicas:1.532064692570853

  [     2 ms] Stats for DiskCapacityGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     985.028 potentialNwOut:     556.011 replicas:53 leaderReplicas:27 topicReplicas:22}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:       0.000 potentialNwOut:       0.000 replicas:43 leaderReplicas:22 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:     464.343 potentialNwOut:     262.106 replicas:4.714045207910317 leaderReplicas:2.1602468994692865 topicReplicas:1.532064692570853

  [     2 ms] Stats for NetworkInboundCapacityGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     985.028 potentialNwOut:     556.011 replicas:53 leaderReplicas:27 topicReplicas:22}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:       0.000 potentialNwOut:       0.000 replicas:43 leaderReplicas:22 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:     464.343 potentialNwOut:     262.106 replicas:4.714045207910317 leaderReplicas:2.1602468994692865 topicReplicas:1.532064692570853

  [     1 ms] Stats for NetworkOutboundCapacityGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     985.028 potentialNwOut:     556.011 replicas:53 leaderReplicas:27 topicReplicas:22}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:       0.000 potentialNwOut:       0.000 replicas:43 leaderReplicas:22 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:     464.343 potentialNwOut:     262.106 replicas:4.714045207910317 leaderReplicas:2.1602468994692865 topicReplicas:1.532064692570853

  [     1 ms] Stats for CpuCapacityGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     985.028 potentialNwOut:     556.011 replicas:53 leaderReplicas:27 topicReplicas:22}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:       0.000 potentialNwOut:       0.000 replicas:43 leaderReplicas:22 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:     464.343 potentialNwOut:     262.106 replicas:4.714045207910317 leaderReplicas:2.1602468994692865 topicReplicas:1.532064692570853

  [     3 ms] Stats for ReplicaDistributionGoal(FIXED):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     985.028 potentialNwOut:     556.011 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:       0.000 potentialNwOut:       0.000 replicas:45 leaderReplicas:22 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:     464.343 potentialNwOut:     262.106 replicas:3.2998316455372216 leaderReplicas:2.1602468994692865 topicReplicas:1.8438694748020148

  [     1 ms] Stats for PotentialNwOutGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     985.028 potentialNwOut:     556.011 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:       0.000 potentialNwOut:       0.000 replicas:45 leaderReplicas:22 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:     464.343 potentialNwOut:     262.106 replicas:3.2998316455372216 leaderReplicas:2.1602468994692865 topicReplicas:1.8438694748020148

  [     5 ms] Stats for DiskUsageDistributionGoal(VIOLATED):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     687.773 potentialNwOut:     403.316 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     595.351 potentialNwOut:     333.606 replicas:48 leaderReplicas:23 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:      43.368 potentialNwOut:      28.630 replicas:1.699673171197595 leaderReplicas:1.632993161855452 topicReplicas:0.7832093030221935

  [     1 ms] Stats for NetworkInboundUsageDistributionGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     687.773 potentialNwOut:     403.316 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     595.351 potentialNwOut:     333.606 replicas:48 leaderReplicas:23 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:      43.368 potentialNwOut:      28.630 replicas:1.699673171197595 leaderReplicas:1.632993161855452 topicReplicas:0.7832093030221935

  [     1 ms] Stats for NetworkOutboundUsageDistributionGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     687.773 potentialNwOut:     403.316 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     595.351 potentialNwOut:     333.606 replicas:48 leaderReplicas:23 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:      43.368 potentialNwOut:      28.630 replicas:1.699673171197595 leaderReplicas:1.632993161855452 topicReplicas:0.7832093030221935

  [     2 ms] Stats for CpuUsageDistributionGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     687.773 potentialNwOut:     403.316 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     595.351 potentialNwOut:     333.606 replicas:48 leaderReplicas:23 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:      43.368 potentialNwOut:      28.630 replicas:1.699673171197595 leaderReplicas:1.632993161855452 topicReplicas:0.7832093030221935

  [     1 ms] Stats for TopicReplicaDistributionGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     687.773 potentialNwOut:     403.316 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     595.351 potentialNwOut:     333.606 replicas:48 leaderReplicas:23 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:      43.368 potentialNwOut:      28.630 replicas:1.699673171197595 leaderReplicas:1.632993161855452 topicReplicas:0.7832093030221935

  [     1 ms] Stats for LeaderReplicaDistributionGoal(NO-ACTION):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     687.773 potentialNwOut:     403.316 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     595.351 potentialNwOut:     333.606 replicas:48 leaderReplicas:23 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:      43.368 potentialNwOut:      28.630 replicas:1.699673171197595 leaderReplicas:1.632993161855452 topicReplicas:0.7832093030221935

  [    11 ms] Stats for LeaderBytesInDistributionGoal(VIOLATED):
  AVG:{cpu:       0.000 networkInbound:     370.684 networkOutbound:     185.337 disk:     656.680 potentialNwOut:     370.674 replicas:49.666666666666664 leaderReplicas:25.0 topicReplicas:12.416666666666666}
  MAX:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     687.773 potentialNwOut:     403.316 replicas:52 leaderReplicas:27 topicReplicas:23}
  MIN:{cpu:       0.000 networkInbound:    1112.053 networkOutbound:     556.011 disk:     595.351 potentialNwOut:     333.606 replicas:48 leaderReplicas:23 topicReplicas:0}
  STD:{cpu:       0.000 networkInbound:       0.000 networkOutbound:       0.000 disk:      43.368 potentialNwOut:      28.630 replicas:1.699673171197595 leaderReplicas:1.632993161855452 topicReplicas:0.7832093030221935

  Cluster load after adding broker [2]:


       HOST         BROKER      RACK         DISK_CAP(MB)            DISK(MB)/_(%)_            CORE_NUM         CPU(%)          NW_IN_CAP(KB/s)       LEADER_NW_IN(KB/s)     FOLLOWER_NW_IN(KB/s)         NW_OUT_CAP(KB/s)        NW_OUT(KB/s)       PNW_OUT(KB/s)    LEADERS/REPLICAS
  localhost,             0,localhost,           10000.000,            686.917/06.87,                  1,         0.000,               10000.000,                 209.129,                 165.970,               10000.000,            209.129,            375.099,            23/48
  localhost,             1,localhost,           10000.000,            687.773/06.88,                  1,         0.000,               10000.000,                 124.477,                 278.839,               10000.000,            124.477,            403.316,            25/49
  localhost,             2,localhost,           10000.000,            595.351/05.95,                  1,         0.000,               10000.000,                 222.436,                 111.202,               10000.000,            222.404,            333.606,            27/52
  * Connection #0 to host localhost left intact
  ```
* if your found error 500 after call proposals such as
  ```bash
  Error processing POST request '/add_broker' due to: 'com.linkedin.kafka.cruisecontrol.exception.KafkaCruiseControlException: com.linkedin.cruisecontrol.exception.NotEnoughValidWindowsException: There are only 0 valid windows when aggregating in range [-1, 1670319259389] for aggregation options (minValidEntityRatio=0.95, minValidEntityGroupRatio=0.00, minValidWindows=1, numEntitiesToInclude=75, granularity=ENTITY)'
  ```

  please wait and re try again because Cruise Control has not collect enough metric data to aggregate to a valid window. (around 2-3 minutes)

* Approving an optimization proposal
  ```bash
  curl -v -X POST 'localhost:9191/kafkacruisecontrol/add_broker?dryrun=false&brokerid=2'
  ```
* Check progress
  ```bash
  curl 'localhost:9191/kafkacruisecontrol/user_tasks'
  ```  
  example result
  ```bash
  USER TASK ID                          CLIENT ADDRESS        START TIME            STATUS       REQUEST URL
  081d2454-5c1c-4c9f-be3c-90d282fdf0fe  [0:0:0:0:0:0:0:1]     2022-11-24T06:20:30Z  InExecution  POST /kafkacruisecontrol/add_broker?brokerid=2&dryrun=false
  7bb50955-f9e5-45e6-bb6b-6177d2b3dc8e  [0:0:0:0:0:0:0:1]     2022-11-24T06:13:33Z  Completed    GET /kafkacruisecontrol/state
  5c0d11d0-0b98-431b-965d-f030e3263835  [0:0:0:0:0:0:0:1]     2022-11-24T06:13:38Z  Completed    GET /kafkacruisecontrol/state
  ```
  call check until status change to Completed
  ```bash
  USER TASK ID                          CLIENT ADDRESS        START TIME            STATUS      REQUEST URL
  7bb50955-f9e5-45e6-bb6b-6177d2b3dc8e  [0:0:0:0:0:0:0:1]     2022-11-24T06:13:33Z  Completed   GET /kafkacruisecontrol/state
  ...
  081d2454-5c1c-4c9f-be3c-90d282fdf0fe  [0:0:0:0:0:0:0:1]     2022-11-24T06:20:30Z  Completed   POST /kafkacruisecontrol/add_broker?brokerid=2&dryrun=false
  ```
* Check Topic after call cruise-control/add_broker, some partition move to new broker

  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
  ```
  example result, show partition of demo topic in broker 0,broker 1 and broker 2
  ```bash
  Topic: demo     TopicId: Z5cEJOZdQ_iUsuzXOWvdfw PartitionCount: 10      ReplicationFactor: 2    Configs: segment.bytes=104857600
        Topic: demo     Partition: 0    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: demo     Partition: 1    Leader: 2       Replicas: 2,1   Isr: 1,2
        Topic: demo     Partition: 2    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: demo     Partition: 3    Leader: 2       Replicas: 2,1   Isr: 1,2
        Topic: demo     Partition: 4    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: demo     Partition: 5    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: demo     Partition: 6    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: demo     Partition: 7    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: demo     Partition: 8    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: demo     Partition: 9    Leader: 0       Replicas: 0,2   Isr: 0,2
  ```

## Stop Server
* stop cruise-control
  ```bash
  cd ~/amq-streams-2022/4-management/cruise-control
  ./kafka-cruise-control-stop.sh config/cruisecontrol.properties 9191
  ```
* stop kafka broker in each terminal with ctrl+c
* stop zookeeper in each terminal with ctrl+c

