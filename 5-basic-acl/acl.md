# Access Control List

<!-- TOC -->

- [Access Control List](#access-control-list)
  - [Prerequisite](#prerequisite)
  - [Create Java Authentication and Authorization Service (JAAS) for Kafa Authentication](#create-java-authentication-and-authorization-service-jaas-for-kafa-authentication)
  - [Set Kafka Broker Authentication](#set-kafka-broker-authentication)
  - [Try Authentication to Kafka](#try-authentication-to-kafka)
  - [Create Client-Site Setting](#create-client-site-setting)
  - [Adding ACL rules](#adding-acl-rules)
  - [Add acl for consume topic](#add-acl-for-consume-topic)
  - [Add authorized for Describe Consumer Group](#add-authorized-for-describe-consumer-group)
  - [List current ACL](#list-current-acl)

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

## Create Java Authentication and Authorization Service (JAAS) for Kafa Authentication
* Configure the broker with its user credentials and authorize the client's user credentials. These credentials along with the login module specification, are stored in a JAAS login configuration file , config JAAS at [jaas.conf](kafka/config/jaas.conf)
  
  ```conf
  KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="admin"
    user_admin="admin"
    user_alice="alice"
    user_bob="bob"
    user_charlie="charlie";
  };  
  ```
  
  This example defines the following for the KafkaServer entity:

  - The custom login module that is used for user authentication,
  - admin/admin is the username and password for inter-broker communication (i.e. the credentials the broker uses to connect to other brokers in the cluster),
  - admin/admin, alice/alice, bob/bob, and charlie/charlie as client user credentials. Note that the valid username and password is provided in this format: user_username="password". If the line user_admin="admin" is removed from this file, the broker is not able to authenticate and authorize an admin user. Only the admin user can to connect to other brokers in this case.

* Pass in this file as a JVM configuration option when running the broker, using -Djava.security.auth.login.config=[path_to_jaas_file]. [path_to_jaas_file] can be something like: config/jaas-kafka-server.conf. This can be done by setting the KAFKA_OPTS environment variable, for example:
  
  ```bash
  export KAFKA_OPTS="-Djava.security.auth.login.config=<path-to-jaas-file>/jaas-kafka-server.conf"
  ```
  
  - run below command to add jaas to "KAFKA_OPTS" environment variable
  
  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  export KAFKA_OPTS="-Djava.security.auth.login.config=~/amq-streams-2022/5-basic-acl/kafka/config/jaas.conf"
  ```

## Set Kafka Broker Authentication
* Define the accepted protocol and the ACL authorizer used by the broker by adding the following configuration to the broker properties file [server.properties](kafka/config/server.properties)
  
  ```properties
  authorizer.class.name=kafka.security.authorizer.AclAuthorizer
  listeners=SASL_PLAINTEXT://:9092
  security.inter.broker.protocol= SASL_PLAINTEXT
  sasl.mechanism.inter.broker.protocol=PLAIN
  sasl.enabled.mechanisms=PLAIN
  ```

* The other configuration that can be added is for Kafka super users: users with full access to all APIs. This configuration reduces the overhead of defining per-API ACLs for the user who is meant to have full API access. From our list of users, let's make admin a super user with the following configuration:

  ```properties
  super.users=User:admin
  ```

* When the broker runs with this security configuration, only authenticated and authorized clients are able to connect to and use it.
* review [server.properties](./kafka/config/server.properties) in folder 5-basic-acl/kafka/bin/config/server.properties

## Try Authentication to Kafka
* start zookeeper, open new terminal and run below command to start zookeeper

  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  ./kafka/bin/zookeeper-server-start.sh ./kafka/config/zookeeper.properties
  ```

* start kafka, open new terminal and run below command to start kafka

  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  export KAFKA_OPTS="-Djava.security.auth.login.config=~/amq-streams-2022/5-basic-acl/kafka/config/jaas.conf"
  ./kafka/bin/kafka-server-start.sh ./kafka/config/server.properties
  ```

* try to get list of topic, open new terminal and run below command
  
  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```

  example result, So far, the broker is configured for authenticated access. Running a Kafka console producer or consumer not configured for authenticated and authorized access fails with messages like the following 

  ```bash
  Error while executing topic command : Timed out waiting for a node assignment. Call: listTopics
  [2022-11-30 15:41:47,753] ERROR org.apache.kafka.common.errors.TimeoutException: Timed out waiting for a node assignment. Call: listTopics
  (kafka.admin.TopicCommand$)
  ```
  
  error result in kafka console

  ```bash
  [2022-11-30 15:41:46,878] INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Failed authentication with /127.0.0.1 (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)
  [2022-11-30 15:41:47,281] INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Failed authentication with /127.0.0.1 (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)
  [2022-11-30 15:41:47,684] INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Failed authentication with /127.0.0.1 (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)
  ```
 
## Create Client-Site Setting
* Specify the broker protocol as well as the credentials to use on the client side. The following configuration is placed inside the corresponding configuration file (admin.properties) provided to the particular client, review following properties in kafka/config folder
  - [admin.properties](kafka/config/admin.properties)

  ```properties
  security.protocol=SASL_PLAINTEXT
  sasl.mechanism=PLAIN
  sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin";
  ```
  
  - [alice.properties](kafka/config/alice.properties)

  ```properties
  security.protocol=SASL_PLAINTEXT
  sasl.mechanism=PLAIN
  sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="alice" password="alice";
  ```
  
  - [bob.properties](kafka/config/bob.properties)

  ```properties
  security.protocol=SASL_PLAINTEXT
  sasl.mechanism=PLAIN
  sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="bob" password="bob";
  ```
  
  - [charlie.properties](kafka/config/charlie.properties)

  ```properties
  security.protocol=SASL_PLAINTEXT
  sasl.mechanism=PLAIN
  sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="charlie" password="charlie";
  ``` 

* Try list topic again with admin user

  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list --command-config kafka/config/admin.properties
  ```
  
  Verify that it can be called without error.
  
* create topic "test"
  
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test --partitions 1 --replication-factor 1 --command-config kafka/config/admin.properties
  ```
  
* List the topics.
  
  ```bash
  ./kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list --command-config kafka/config/admin.properties
  ```

* Try to put data to topic "test" with user:alice
  
  ```bash
  cd ~/amq-streams-2022/5-basic-acl
  ./kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
  ```
  
  example result, ctrl+c to exit

  ```bash
  [2022-12-02 11:33:28,628] WARN [Producer clientId=console-producer] Bootstrap broker localhost:9092 (id: -1 rack: null) disconnected (org.apache.kafka.clients.NetworkClient)
  [2022-12-02 11:33:29,036] WARN [Producer clientId=console-producer] Bootstrap broker localhost:9092 (id: -1 rack: null) disconnected (org.apache.kafka.clients.NetworkClient)
  [2022-12-02 11:33:29,439] WARN [Producer clientId=console-producer] Bootstrap broker localhost:9092 (id: -1 rack: null) disconnected (org.apache.kafka.clients.NetworkClient)
  ```
  
  error result in kafka console

  ```bash
  [2022-11-30 15:41:46,878] INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Failed authentication with /127.0.0.1 (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)
  [2022-11-30 15:41:47,281] INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Failed authentication with /127.0.0.1 (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)
  [2022-11-30 15:41:47,684] INFO [SocketServer listenerType=ZK_BROKER, nodeId=0] Failed authentication with /127.0.0.1 (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)
  ```
  
* Test Again with alice user
  
  ```bash
  ./kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test --producer.config kafka/config/alice.properties
  ```

  example result
  ```bash
  >
  ```

  try to send message such as "message1"
  ```bash
  >message1
  ```
  
  example result, Authorization Exception will be throw to your console.

  ```bash
  >message1
  [2022-12-02 11:39:04,065] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 3 : {test=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
  [2022-12-02 11:39:04,068] ERROR [Producer clientId=console-producer] Topic authorization failed for topics [test] (org.apache.kafka.clients.Metadata)
  [2022-12-02 11:39:04,069] ERROR Error when sending message to topic test with key: null, value: 8 bytes with error: (org.apache.kafka.clients.producer.internals.ErrorLoggingCallback)
  org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [test]
  >
  ```

  exit from producer console by type ctrl+c.

## Adding ACL rules

* Check current access control list
  
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --list --command-config kafka/config/admin.properties 
  ```
  
  example result, no acl
  ```bash

  ```

* add authorized to alice user
  
  requirement are:
  
  ```bash
  Principal alice is Allowed Operation Write From Host * On Topic test.
  Principal alice is Allowed Operation Create From Host * On Topic test.
  ```
  
  run command to add authorized
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config kafka/config/admin.properties --add --allow-principal User:alice --operation Write --operation Create --topic test
  ```
  
  example result
  ```
  Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
        (principal=User:alice, host=*, operation=WRITE, permissionType=ALLOW)
        (principal=User:alice, host=*, operation=CREATE, permissionType=ALLOW) 

  Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
          (principal=User:alice, host=*, operation=CREATE, permissionType=ALLOW)
          (principal=User:alice, host=*, operation=WRITE, permissionType=ALLOW) 
  ```
  
  
* Test Again with alice user
  
  ```bash
  ./kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test --producer.config kafka/config/alice.properties
  ```
  
  example result, try to send message to topic "test"
  ```bash
  >message1
  >message2
  >message3
  >
  ```

* exit from kafka console producer and check current acl
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --list --command-config kafka/config/admin.properties 
  ```

  example result
  ```bash
  Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
        (principal=User:alice, host=*, operation=CREATE, permissionType=ALLOW)
        (principal=User:alice, host=*, operation=WRITE, permissionType=ALLOW)
  ```

## Add acl for consume topic 

* test consume message with user: bob
  ```bash
  ./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --group bob-group --consumer.config kafka/config/bob.properties --from-beginning
  ```

  example result, error is
  ```bash
  [2022-12-02 12:03:00,033] WARN [Consumer clientId=console-consumer, groupId=bob-group] Error while fetching metadata with correlation id 2 : {test=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
  [2022-12-02 12:03:00,034] ERROR [Consumer clientId=console-consumer, groupId=bob-group] Topic authorization failed for topics [test] (org.apache.kafka.clients.Metadata)
  [2022-12-02 12:03:00,035] ERROR Error processing message, terminating consumer process:  (kafka.tools.ConsoleConsumer$)
  org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [test]
  Processed a total of 0 messages
  ```

* add read authorized
  
  requirement are:
  
  ```bash
  Principal bob is Allowed Operation Read From Host * On Topic test.
  ```
  
  run command to add authorized
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config kafka/config/admin.properties --add --allow-principal User:bob --operation Read --topic test
  ```
  
  example result
  ```bash
  Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
        (principal=User:bob, host=*, operation=READ, permissionType=ALLOW) 

  Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
          (principal=User:alice, host=*, operation=CREATE, permissionType=ALLOW)
          (principal=User:bob, host=*, operation=READ, permissionType=ALLOW)
          (principal=User:alice, host=*, operation=WRITE, permissionType=ALLOW)
  ```

* add authorized for committing offsets to group bob-group  
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config kafka/config/admin.properties --add --allow-principal User:bob --operation Read --group bob-group
  ```
  
  example result
  ```bash
  Adding ACLs for resource `ResourcePattern(resourceType=GROUP, name=bob-group, patternType=LITERAL)`: 
        (principal=User:bob, host=*, operation=READ, permissionType=ALLOW) 

  Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=bob-group, patternType=LITERAL)`: 
          (principal=User:bob, host=*, operation=READ, permissionType=ALLOW)
  ```
  
* test consume message again
  ```bash
  ./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --group bob-group --consumer.config kafka/config/bob.properties --from-beginning
  ```

  example result
  ```bash
  message1
  message2
  message3
  ```

## Add authorized for Describe Consumer Group
* add authorized for describe consumer group to user: charlie
  
  requirement are
  ```bash
  Principal charlie is Allowed Operation Describe From Host * On Group bob-group.
  Principal charlie is Allowed Operation Describe From Host * On Topic test.
  ```

  run command to add authorized to read group bob-group
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config kafka/config/admin.properties --add --allow-principal User:charlie --operation Read --group bob-group
  ```
  
  example result
  ```bash
  Adding ACLs for resource `ResourcePattern(resourceType=GROUP, name=bob-group, patternType=LITERAL)`: 
        (principal=User:charlie, host=*, operation=READ, permissionType=ALLOW) 

  Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=bob-group, patternType=LITERAL)`: 
          (principal=User:bob, host=*, operation=READ, permissionType=ALLOW)
          (principal=User:charlie, host=*, operation=READ, permissionType=ALLOW)
  ```

  run command to add authorized to describe topic test
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config kafka/config/admin.properties --add --allow-principal User:charlie --operation Describe --topic test
  ```

  example result
  ```bash
  Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
        (principal=User:charlie, host=*, operation=DESCRIBE, permissionType=ALLOW) 

  Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
          (principal=User:alice, host=*, operation=CREATE, permissionType=ALLOW)
          (principal=User:bob, host=*, operation=READ, permissionType=ALLOW)
          (principal=User:charlie, host=*, operation=DESCRIBE, permissionType=ALLOW)
          (principal=User:alice, host=*, operation=WRITE, permissionType=ALLOW)
  ```
* test describe consumer group
  ```bash
  ./kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group bob-group --command-config kafka/config/charlie.properties
  ```

  example result
  ```bash
  Consumer group 'bob-group' has no active members.

  GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
  bob-group       test            0          3               3               0               -               -               -
  ```

## List current ACL
* run command to check current ACL in this kafka cluster
  ```bash
  ./kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config kafka/config/admin.properties --list
  ```

  example result
  ```bash
  Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`: 
        (principal=User:alice, host=*, operation=CREATE, permissionType=ALLOW)
        (principal=User:bob, host=*, operation=READ, permissionType=ALLOW)
        (principal=User:charlie, host=*, operation=DESCRIBE, permissionType=ALLOW)
        (principal=User:alice, host=*, operation=WRITE, permissionType=ALLOW) 

  Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=bob-group, patternType=LITERAL)`: 
          (principal=User:bob, host=*, operation=READ, permissionType=ALLOW)
          (principal=User:charlie, host=*, operation=READ, permissionType=ALLOW) 
  ```