# Introduction to AMQ Streams

## Lab Folder

* Go to Lab Folder

```
cd 1-introduction-amq-streams
```

## Start ZooKeeper

* First before we start Kafka, we have to start ZooKeeper, open new terminal and run command

```
./kafka/bin/zookeeper-server-start.sh ./kafka/config/zookeeper.properties
```

## Start Kafka

* Next we can start Kafka, open new terminal and run command

```
./kafka/bin/kafka-server-start.sh ./kafka/config/server.properties
```

## Create and list topics

* List the topics using Kafka, open new terminal and run command

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

* no topic show in terminal
* Create sample new topic which we will use

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my-topic --partitions 3 --replication-factor 1
```

* List the topics again to see it was created.

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
my-topic
```

* Describe the topic to see more details:

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic my-topic --describe
```

* example output

```
Topic: my-topic	TopicId: CXoa2jvVScmiOx6wWx0VzQ	PartitionCount: 3	ReplicationFactor: 1	Configs: segment.bytes=1073741824
	Topic: my-topic	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	Topic: my-topic	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	Topic: my-topic	Partition: 2	Leader: 0	Replicas: 0	Isr: 0
```

## Consuming and producing messages

* Start the console producer:

```
./kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-topic
```

* Wait until it is ready (it should show `>`).

* Next we can consume the messages, open new terminal and run command

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-topic --from-beginning
```

* Once ready, back to producer terminal and send some message by typing the message payload and pressing enter to send. such as
  
```
>a
>b
>c
>
```

* see output in consumer terminal:
```
a
b
c
```

* You can also check the consumer groups:

```
./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --all-groups
```

