# Consumer and Producer APIs

## Setup

Setup and start the Zookeeper and Kafka cluster from the [Kafka Architecture demo](../kafka-architecture/).
This cluster will be used during this demo. 

## Topic

* Create new topic for out test messages:

```
cd 2-amq-streams-architecture
./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic weather-report --partitions 1 --replication-factor 3
```

## Consumer

* Start a console consumer to see the sent messages:

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic weather-report --from-beginning --property print.key=true --property key.separator=":     "
```

## Consumer and Producer APIs

Open the attached Java project in IDE and go through the files.
You can run them against the brokers.
There are several consumer and producer pair showing different options:

- Super simple basic consumer producer
  
  ```
  cd 3-consumer-producer
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.basic.ProducerAPI"
  ```

  ```
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.basic.ConsumerAPI"
  ```

- Acks and manual commits
- 
  ```
  cd 3-consumer-producer
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.acks.ProducerAPI"
  ```

  ```
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.manualcommits.ConsumerAPI"
  ```

- SSL and SSL Authentication
  
  ```
  cd 3-consumer-producer
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.ssl.ProducerAPI"
  ```

  ```
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.ssl.ConsumerAPI"
  ``` 

- SASL
  
  ```
  cd 3-consumer-producer
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.sasl.ProducerAPI"
  ```

  ```
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.sasl.ConsumerAPI"
  ``` 

- Serialization and deserialization

  ```
  cd 3-consumer-producer
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.serialization.ProducerAPI"
  ```

  ```
  mvn compile exec:java -Dexec.mainClass="cz.scholz.kafka.serialization.ConsumerAPI"
  ``` 