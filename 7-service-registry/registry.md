# Service Registry

<!-- TOC -->

- [Service Registry](#service-registry)
  - [Prerequisite](#prerequisite)
  - [Install Service Registry](#install-service-registry)
    - [Install Service Registry Operator (OpenShift Admin Task, Ready Now for This Lab)](#install-service-registry-operator-openshift-admin-task-ready-now-for-this-lab)
    - [Create OpenShift Project](#create-openshift-project)
    - [Create Database for Service Registry](#create-database-for-service-registry)
    - [Create Service Registry](#create-service-registry)
  - [Managing schema and API artifacts using Service Registry REST API commands](#managing-schema-and-api-artifacts-using-service-registry-rest-api-commands)
  - [Kafka Client Schema Validation with Service Registry](#kafka-client-schema-validation-with-service-registry)
  - [More Example of Service Registry](#more-example-of-service-registry)
  - [More Detail about Advanced Red Hat Service Registry](#more-detail-about-advanced-red-hat-service-registry)

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

* start zookeeper
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/zookeeper-server-start.sh ./kafka/config/zookeeper.properties
  ```

* start kafka broker
  ```bash
  cd ~/amq-streams-2022/4-management
  ./kafka/bin/kafka-server-start.sh ./kafka/config/server.properties
  ```

## Install Service Registry
### Install Service Registry Operator (OpenShift Admin Task, Ready Now for This Lab)
* Login to red hat openshift with admin user
* In Administrator Perspective, Go to Operators -> OperatorHub menu. Enter service registry into the search box, the Red Hat Integration - Service Registry Operator will show up on the screen. Then click on it.
  
  ![](./../images/registry-1.png)
  
* A panel with details of the operator will show up on the right. Then click Install button.
  
  ![](./../images/registry-2.png)
  
* You can leave all options as default or change them if needed i.e. install the operator to the project you've created earlier. Then click Install button.

  ![](./../images/registry-3.png)

* Wait until the operator gets installed successfully before proceeding to next steps.
  
  ![](./../images/registry-4.png)


### Create OpenShift Project
* Open your web browser, Login to red hat openshift with 
  - Openshift Console URL : --> get it from your instructor
  - User Name/Password : --> get it from your instructor such as user1/openshift
  - go to developer console
    
     ![](./../images/registry-5.png)
    
* Go to Projects menu, then click on Create Project button.
  
  ![](./../images/registry-6.png)
  
* Enter a project name such as YOUR_USER_NAME-service-registry (such as user1-service-registry) 
  
  ![](./../images/registry-7.png)
  
* Change YOUR_USER_NAME to your user name then click on Create button.
  
  ![](./../images/registry-8.png)

### Create Database for Service Registry
* Service Registry support 2 type of storage, kafka & database. we select database for this lab.
* In Developer Console, select Menu Topology from left menu. right click on panel and select Add to Project --> Database, Console will change to Database Catalog

  ![](./../images/registry-9.png)

* In Database Catalog, select PostgreSQL

  ![](./../images/registry-10.png)

* Click Instantiate Template
  
  ![](./../images/registry-11.png)

* In Instantiate Template, leave all default value except:
  - Database Service Name : service-registry
  - PostgreSQL Conneciton Username : service-registry
  - PostgreSQL Connection Password : service-registry
  - PostgreSQL Database Name : service-registry
    
  ![](./../images/registry-12.png)

* click Create and wait until database ready (circle around pod service-registry change to dark blue color)

  ![](./../images/registry-13.png)

### Create Service Registry
* In Topology View, click Add to Project Button (book icon)
 
  ![](./../images/registry-14.png)
   
* type "registry" in search box, select "Apicurio Registry" and click Create button
    
  ![](./../images/registry-15.png)

* In Create ApicurioRegistry Panel, change configure via to YAML view and copy yaml from [apicurio.yml](apicurio.yml) to editor
  
  ![](./../images/registry-16.png)

  - check username/password is "service-registry"
  - check jdc url, change user1-service-registry to your project name
  - click "Create" button

* wait until service registry ready (change color to dark blue)  
  
  ![](./../images/registry-17.png)

* open service registry console by click Open URL on service registry poc icon
    
  ![](./../images/registry-18.png)
  
  ![](./../images/registry-19.png)

## Managing schema and API artifacts using Service Registry REST API commands
* create schema on service registry with REST API command line
  
  - chnage "user1-service-registry" to your project name before run below command!
  - change "apps.cluster-82xm2.82xm2.sandbox503.opentlc.com" to your current domain of openshift --> get it from openshift console url
  
  ```bash
  curl -X POST -H "Content-Type: application/json; artifactType=AVRO" \
  -H "X-Registry-ArtifactId: share-price" \
  --data '{"type":"record","name":"price","namespace":"com.example", \
   "fields":[{"name":"symbol","type":"string"},{"name":"price","type":"string"}]}' \
  http://service-registry.user1-service-registry.router-default.apps.cluster-82xm2.82xm2.sandbox503.opentlc.com/apis/registry/v2/groups/my-group/artifacts
  ```

  example result
  ```bash
  {"createdBy":"","createdOn":"2022-12-08T04:36:16+0000","modifiedBy":"","modifiedOn":"2022-12-08T04:36:16+0000","id":"share-price","version":"1","type":"AVRO","globalId":1,"state":"ENABLED","groupId":"my-group","contentId":1,"references":[]}
  ```

* retreive schama from service registry with REST API command line
  
  - change "user1-service-registry" to your project name before run below command!
  - change "apps.cluster-82xm2.82xm2.sandbox503.opentlc.com" to your current domain of openshift --> get it from openshift console url
  
  ```bash
  curl http://service-registry.user1-service-registry.router-default.apps.cluster-82xm2.82xm2.sandbox503.opentlc.com/apis/registry/v2/groups/my-group/artifacts/share-price 
  ```
  
  example result
  
  ```bash
  {"type":"record","name":"price","namespace":"com.example", "fields":[{"name":"symbol","type":"string"},{"name":"price","type":"string"}]}
  ```

* Review previous schema in service registry console

  ![](./../images/registry-20.png)

  ![](./../images/registry-21.png)

  ![](./../images/registry-22.png)

## Kafka Client Schema Validation with Service Registry
* Service Registry provides client serializers/deserializers (SerDes) for Kafka producer and consumer applications written in Java. Kafka producer applications use serializers to encode messages that conform to a specific event schema. Kafka consumer applications use deserializers to validate that messages have been serialized using the correct schema, based on a specific schema ID. This ensures consistent schema use and helps to prevent data errors at runtime.
  
  ![](../images/registry-23.png)
  
* Review Kafka Client Schema Validation Java Client code at [SimpleAvroExample.java](apicurio-registry-examples/simple-avro/src/main/java/io/apicurio/registry/examples/simple/avro/SimpleAvroExample.java)
  - Review and Edit properties of application
    - REREGISTRY_URL : change to your service registry URL
      - change "user1-service-registry" to your project name before run below command!
      - change "apps.cluster-82xm2.82xm2.sandbox503.opentlc.com" to your current domain of openshift --> get it from openshift console url
    - SERVERS : kafka bootstrap such as localhost:9092
    - TOPIC_NAME : topic for this application
    - SCHEMA : ARVO schama of this application

    ```java        
    private static final String REGISTRY_URL = "http://service-registry.user1-service-registry.router-default.apps.cluster-82xm2.82xm2.sandbox503.opentlc.com/apis/registry/v2";
    private static final String SERVERS = "localhost:9092";
    private static final String TOPIC_NAME = SimpleAvroExample.class.getSimpleName();
    private static final String SUBJECT_NAME = "Greeting";
    private static final String SCHEMA = "{\"type\":\"record\",\"name\":\"Greeting\",\"fields\":[{\"name\":\"Message\",\"type\":\"string\"},{\"name\":\"Time\",\"type\":\"long\"}]}";

    ```
  - Review properties setting in createKafkaProducer method
    - KEY_SERIALIZER_CLASS_CONFIG : Serializer for "key" of message
    - VALUE_SERIALIZER_CLASS_CONFIG : Serializer for "message"
    - REGISTRY_URL : URL of Service Registry
    - AUTO_REGISTER_ARTIFACT : set auto register schema to service registry if don't found 
    ```java
    // Configure kafka settings
    props.putIfAbsent(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, SERVERS);
    props.putIfAbsent(ProducerConfig.CLIENT_ID_CONFIG, "Producer-" + TOPIC_NAME);
    props.putIfAbsent(ProducerConfig.ACKS_CONFIG, "all");
    props.putIfAbsent(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
    // Use the Apicurio Registry provided Kafka Serializer for Avro
    props.putIfAbsent(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, AvroKafkaSerializer.class.getName());

    // Configure Service Registry location
    props.putIfAbsent(SerdeConfig.REGISTRY_URL, REGISTRY_URL);
    // Register the artifact if not found in the registry.
    props.putIfAbsent(SerdeConfig.AUTO_REGISTER_ARTIFACT, Boolean.TRUE);

    //Just if security values are present, then we configure them.
    configureSecurityIfPresent(props);
    ```
    
  - Review properties setting in createKafkaConsumer method
    - KEY_DESERIALIZER_CLASS_CONFIG : DeSerializer for "key" of message
    - VALUE_DESERIALIZER_CLASS_CONFIG : DeSerializer for "message"
    - REGISTRY_URL : URL of Service Registry
    ```java
    // Configure Kafka
    props.putIfAbsent(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, SERVERS);
    props.putIfAbsent(ConsumerConfig.GROUP_ID_CONFIG, "Consumer-" + TOPIC_NAME);
    props.putIfAbsent(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
    props.putIfAbsent(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
    props.putIfAbsent(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
    props.putIfAbsent(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
    // Use the Apicurio Registry provided Kafka Deserializer for Avro
    props.putIfAbsent(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, AvroKafkaDeserializer.class.getName());

    // Configure Service Registry location
    props.putIfAbsent(SerdeConfig.REGISTRY_URL, REGISTRY_URL);
    ```

* Try to run kafka client
  ```bash
  cd ~/amq-streams-2022/7-service-registry/apicurio-registry-examples/simple-avro 
  mvn compile exec:java -Dexec.mainClass="io.apicurio.registry.examples.simple.avro.SimpleAvroExample"
  ```

  example result
  ```bash
  Starting example SimpleAvroExample
  SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
  SLF4J: Defaulting to no-operation (NOP) logger implementation
  SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
  Producing (5) messages.
  Messages successfully produced.
  Closing the producer.
  Creating the consumer.
  Subscribing to topic SimpleAvroExample
  Consuming (5) messages.
  Consumed a message: Hello (0)! @ Thu Dec 08 13:32:46 ICT 2022
  Consumed a message: Hello (1)! @ Thu Dec 08 13:32:47 ICT 2022
  Consumed a message: Hello (2)! @ Thu Dec 08 13:32:47 ICT 2022
  Consumed a message: Hello (3)! @ Thu Dec 08 13:32:47 ICT 2022
  Consumed a message: Hello (4)! @ Thu Dec 08 13:32:48 ICT 2022
  Done (success).
  ```

* Try again with invalid schema
  * change code in [SimpleAvroExample.java](apicurio-registry-examples/simple-avro/src/main/java/io/apicurio/registry/examples/simple/avro/SimpleAvroExample.java)
  * go to line 89, remove remark code to enable invalid schema
    
    ```java
                record.put("Message", message);
                record.put("Time", now.getTime());
                //record.put("invalid", "invalid");
    ```

    to

    ```java
                record.put("Message", message);
                record.put("Time", now.getTime());
                record.put("invalid", "invalid");
    ```    
 
  * run client again
    ```bash
    cd ~/amq-streams-2022/7-service-registry/apicurio-registry-examples/simple-avro 
    mvn compile exec:java -Dexec.mainClass="io.apicurio.registry.examples.simple.avro.SimpleAvroExample"
    ```
    
    example result
    ```bash
    Producing (5) messages.
    Closing the producer.
    [WARNING] 
    org.apache.avro.AvroRuntimeException: Not a valid schema field: invalid
        at org.apache.avro.generic.GenericData$Record.put (GenericData.java:253)
        at io.apicurio.registry.examples.simple.avro.SimpleAvroExample.main (SimpleAvroExample.java:89)
    ```


## More [Example](https://github.com/Apicurio/apicurio-registry-examples) of Service Registry
## More Detail about [Advanced Red Hat Service Registry](https://audomsak.gitbook.io/red-hat-service-registry/)  