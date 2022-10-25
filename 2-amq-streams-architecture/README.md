# Kafka Architecture

## Setup

### SSL

First we need to generate the certificates.
For that you need to have the following things installed:
- OpenSSL
  - macos 
    ```
    brew update && brew install openssl
    ```
  - windows
   
- CFSSL
  - macos
    ```
    brew install cfssl
    ```
- Java (keytool), JDK 11

### Clean Kafka Data

```
rm -rf /tmp/zookeeper*
rm -rf /tmp/kafka*
```

### Generate the certificates by running:

```
cd 2-amq-streams-architecture
./ssl/generate.sh
```

### Start ZooKeeper cluster

First before we start Kafka, we have to start ZooKeeper cluster.
We will use 3 node cluster.
Start the 3 Zookeeper nodes by running these 3 scripts in different terminals.
Don't worry about error after run zookeeper-0.sh (error about QuorumPeer):

```
./scripts/zookeeper-0.sh
./scripts/zookeeper-1.sh
./scripts/zookeeper-2.sh
```

### Start Kafka cluster

We will use 3 node Kafka cluster
Start the 3 Kafka nodes by running these 3 scripts in different terminals:

```
./scripts/kafka-0.sh
./scripts/kafka-1.sh
./scripts/kafka-2.sh
```

## Configs

### Zookeeper configuration

* Look at the Zookeeper config files in `./configs/zookeeper/`
* Look at the ensemble configuration in the Zookeeper properties files
* Check the content of the Zookeeper data dirs in `/tmp`
    * Notice the `myid` file which needs to be created before starting Zookeeper with the node ID

### Kafka configuration

* Look at the Kafka configuration files in `./configs/kafka/`
* Notice:
    * `broker.id`
    * listeners, advertised listeners, protocols
    * Zookeeper config
* Look at the data dir in `/tmp`
* Look at the tools in `./kafka/bin`

## Zookeeper

### Show what Kafka does in Zookeeper

* Find and notice the ZK JAR files in `./kafka/libs` and `./kafka/bin` - Zookeeper is integrated into Kafka distribution
* Start the ZK client

```
./kafka/bin/zookeeper-shell.sh localhost:2181
```

* Browse through the Zookeeper nodes
    * You can for example investigate the following nodes:

```
ls /
get /controller
ls /brokers
ls /brokers/ids
get /brokers/ids/0
ls /brokers/topics
```

* Exit from zookeeper-shell with ctrl+c
* Try to do netcat dump with connected brokers

```
echo dump | nc localhost 2181
```

* Kill broker 2 (with ctrl+c in broker 2 terminal) and do the netcat again to see how it disappeared

```
echo dump | nc localhost 2181
```

* start broker 2 again

## Basics

### Create topic

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic demo --partitions 3 --replication-factor 3
```

### Check the created topic

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
```

Notice the distribution of leaders and the ISR replicas.

### Send some messages

* Send at least 10 messages (e.g. `Message 1`, `Message 2` etc. to be able to notice the ordering later)

```
./kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic demo
```

### Consume messages

* Read from the whole topic

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning
```

* Notice how the messages are out of order. And check how nicely ordered they are in a single partition.

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --partition 0 --from-beginning
```

* Reading from a particular offset

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --partition 0 --offset 2
```

## Replication

### Broker crash

* View topic description with the leaders  and new ISR

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
```

* Kill broker 2
* Look again at the topic description with the leaders which changed and new ISR

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
```

### Consume messages

* Try to consume the messages again to confirm that replication worked and that the messages are still in the topic!

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning
```

### Send some new messages

```
./kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic demo
```

### Start the broker 2 again

* Look again at the topic description with the leaders which changed and new ISR

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
```

* Leadership didn't changed, but all replicas are again ISR

## Consumer Groups

### Create a new topic to get rid of the old messages

```
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic demo
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic demo --partitions 3 --replication-factor 3
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic demo
```

### Setup consumers

_(in different terminals)_

* Open 3 consumers using the same group `group-1`

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning --property print.key=true --property key.separator=":" --group group-1
```

* Open consumer using a different group `group-2`

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning  --property print.key=true --property key.separator=":" --group group-2
```

### Send messages

* Send some messages with keys (Send messages in the format `key:payload` - e.g. `my-key:my-value`)

```
./kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic demo --property "parse.key=true" --property "key.separator=:"
```

### Rebalancing consumer group

* Kill one of the consumers started before
* Send some messages with the same key as was used before for this consumer
* Notice that one of the other consumers got the partition assigned and will receive it

### Message replay

* Consume the messages from Kafka with a new group:

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning  --property print.key=true --property key.separator=":" --group replay-group
```

* After it consumes all messages, try to restart it to make sure they were all committed - no messages should be received
* List all the groups:

```
./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --list
```

* Or describe them:

```
./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --describe
```

* Go and reset the offset to 0:

```
./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --reset-offsets --to-earliest --group replay-group --topic demo --execute
```

* Try to consume the messages again - you should receive them from the beginning of the topic:

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning  --property print.key=true --property key.separator=":" --group replay-group
```

## Security

### Configuration

* Look in the configuration files of the brokers (`./configs/kafka/`) and check the fields related to the TLS & SASL.

### SSL Consumers and Producers

* Use SSL to producer messages:

```
./kafka/bin/kafka-console-producer.sh --broker-list localhost:19092 \
      --topic demo \
      --producer-property security.protocol=SSL \
      --producer-property ssl.truststore.password=123456 \
      --producer-property ssl.truststore.location=./ssl/keys/truststore \
      --producer-property ssl.keystore.password=123456 \
      --producer-property ssl.keystore.location=./ssl/keys/user1.keystore
```

* And consume them:

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:19092 \
      --topic demo --from-beginning \
      --consumer-property security.protocol=SSL \
      --consumer-property ssl.truststore.password=123456 \
      --consumer-property ssl.truststore.location=./ssl/keys/truststore \
      --consumer-property ssl.keystore.password=123456 \
      --consumer-property ssl.keystore.location=./ssl/keys/user1.keystore
```

### SASL Consumers and Producers

* Check the `sasl-client.properties` file  which configures SASL PLAIN authentication 
* Try to producer some messages:

```
./kafka/bin/kafka-console-producer.sh --broker-list localhost:39092 \
      --topic demo \
      --producer.config sasl-client.properties
```

* And consume them:

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:39092 \
      --topic demo --from-beginning \
      --consumer.config sasl-client.properties
```
