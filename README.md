This purpose of this repo is to demostrate a spring boot application interacting with kafka running as part of Confluent Cloud. This example is developed using the Spring Kafka course, https://developer.confluent.io/courses/spring/apache-kafka-intro/

## Table of Contents

- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Project Setup](#project-setup)
- [Produce Messages](#produce-messages)
- [Process Messages](#process-messages)
- [Consume Messages](#consume-messages)
- [JSON Serialization](#json-serialization)
- [Transactions Support](#transactions-support)

## Getting Started

### Prerequisites

Ensure you have the following installed on your development machine:

- [Java Development Kit (JDK) 17](https://openjdk.java.net/)
- [Apache Maven](https://maven.apache.org/)

### Project Setup

1. Open https://start.spring.io/. If you cannot access this website as part of your Organization policy, download the package spring-ccloud-maven.zip in this folder and skip to step 5.

2. Choose options based on the screenshot listed here ![image](./images/project_settings.png). We recommend to use the values mentioned in the screenshot as it makes it easy to follow the rest of the instructions.

3. Add dependencies based on the screenshot listed here ![image](./images/dependencies.png)

4. Click the Generate button to download the package(zip file) to a folder of your choice. If you used the values shown in the screenshot, the generated filename will be spring-ccloud-maven.zip.  

5. unzip the package to extract the file contents to a folder, spring-ccloud-maven

6. `cd spring-ccloud-maven`

7. Open pom.xml and add the below dependency to the `<dependencies>` section. This library is used to send random quotes as messages to kafka. More info about this library can be found at https://github.com/DiUS/java-faker
```
    <dependency>
        <groupId>com.github.javafaker</groupId>
        <artifactId>javafaker</artifactId>
        <version>1.0.2</version>
    </dependency>
```

8. Open application.properties under src/main/resources. Add the below content to it.
```
# Kafka
spring.kafka.properties.sasl.mechanism=PLAIN
spring.kafka.bootstrap-servers=<TOBEFILLED>
spring.kafka.properties.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<TOBEFILLED>' password='<TOBEFILLED>';
spring.kafka.properties.security.protocol=SASL_SSL

#Producer Properties
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.IntegerSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.client-id=spring-boot-producer

#Consumer Properties
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.LongDeserializer

# User defined
topic.name=quotes
json-topic.name=quotes-json
spring.kafka.consumer.group-id=my-group
```
The values for <TOBEFILLED> will be provided by the workshop instructor/moderator

## Produce Messages

In this module we add logic to produce messages to a kafka topic. 

1. Create a new java class, Producer.java under src/main folder in io.confluent.developer.springccloud package. The location should look like in the screenshot below 

![image](./images/producer_location.png)

Copy the below code to Producer.java

```java
package io.confluent.developer.springccloud;

import java.time.Duration;
import java.util.stream.Stream;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.annotation.Bean;
import org.springframework.context.event.EventListener;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import com.github.javafaker.Faker;

import lombok.RequiredArgsConstructor;
import reactor.core.publisher.Flux;

@RequiredArgsConstructor
@Component
public class Producer {

    private final KafkaTemplate<Integer, String> template;

    Faker faker;

    @Bean
    NewTopic quotes() {
        return TopicBuilder.name("quotes").partitions(6).replicas(3).build();
    }

    @EventListener(ApplicationStartedEvent.class)
    public void generate() {

        String quote;
        faker = Faker.instance();

        quote = faker.hobbit().quote();
        System.out.printf("Sending quote: %s %n", quote);
        template.send("quotes", faker.random().nextInt(42), quote);

    }
}

```

2. Being in the root folder in spring-ccloud-maven, run `mvn package`

3. If you see messages on the console with the prefix `Sending quote: ` followed by a random quote, then the data was successfully published to kafka topics. If you see any errors in the console, please check with the instructor/moderator.

4. Messages should appear in the relevant topic. Open CClould UI, go to the relevant cluster and click on the corresponding topic. Your instructor/moderator will be able to help with this check.

## Process Messages

In this module we are going to use KStream to process the messages that were published as part of Producer in previous module.

1. Create a new java class, Processor.java under src/main folder in io.confluent.developer.springccloud package.

Copy the below code to Processor.java

```java
package io.confluent.developer.springccloud;

import java.util.Arrays;

import org.apache.kafka.clients.admin.NewTopic;
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.Consumed;
import org.apache.kafka.streams.kstream.Grouped;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.kstream.Produced;
import org.apache.kafka.streams.kstream.ValueMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.stereotype.Component;

@Component
public class Processor {

    @Bean
    NewTopic quotesWordCount() {
        return TopicBuilder.name("quotes-wordcount-output").partitions(6).replicas(3).build();
    }

    @Bean
    NewTopic quotesUpperCase() {
        return TopicBuilder.name("quotes-upper-case").partitions(6).replicas(3).build();
    }
    
    @Autowired
    public void process(StreamsBuilder builder) {

        // Serializers/deserializers (serde) for String and Long types
        final Serde<Integer> integerSerde = Serdes.Integer();
        final Serde<String> stringSerde = Serdes.String();
        final Serde<Long> longSerde = Serdes.Long();

        // Construct a `KStream` from the input topic, where
        // message values
        // represent lines of text (for the sake of this example, we ignore whatever may
        // be stored
        // in the message keys).
        KStream<Integer, String> textLines = builder
                .stream("quotes", Consumed.with(integerSerde, stringSerde));

        KStream<Integer, String> upperCased = textLines.mapValues((ValueMapper<String, String>) String::toUpperCase);
        upperCased.to("quotes-upper-case", Produced.with(integerSerde, stringSerde));

        KTable<String, Long> wordCounts = textLines
                // Split each text line, by whitespace, into words. The text lines are the
                // message
                // values, i.e. we can ignore whatever data is in the message keys and thus
                // invoke
                // `flatMapValues` instead of the more generic `flatMap`.
                .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
                // We use `groupBy` to ensure the words are available as message keys
                .groupBy((key, value) -> value, Grouped.with(stringSerde, stringSerde))
                // Count the occurrences of each word (message key).
                .count(Materialized.as("counts"));

        // Convert the `KTable<String, Long>` into a `KStream<String, Long>` and write
        // to the output topic.
        wordCounts.toStream().to("quotes-wordcount-output", Produced.with(stringSerde, longSerde));

    }
}
```

2. Being in the root folder in spring-ccloud-maven, run `mvn package`

3. If everything is succesful, you should see messages being published in quotes-upper-case and quotes-wordcount-output topics. If you cannot access UI, take help from your instructor/moderator. If you do not see this or see any errors, please check with your instructor/moderator.

## Consume Messages

In this module we add logic to consume messages from a kafka topic. 

1. Create a new java class, Consumer.java under src/main folder in io.confluent.developer.springccloud package. The location should look like in the screenshot below 

![image](./images/consumer_location.png)

Copy the below code to Consumer.java

```java
package io.confluent.developer.springccloud;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class Consumer {

    @KafkaListener(topics = { "quotes-wordcount-output" }, groupId = "spring-boot-kafka")
    public void consume(ConsumerRecord<String, Long> record) {
        System.out.println("received = " + record.value() + " with key " + record.key());
    }
}

```

2. Being in the root folder in spring-ccloud-maven, run `mvn package`

3. If everything is succesful, you should see messages in the console with the term `received = `. If you do not see this or see any errors, please check with your instructor/moderator.

4. Once you validate that the messages are successfully consumed you can hit ctrl+c to exit out of the maven process.


## JSON Serialization
In this module we learn how to produce and consume messages in JSON format using JSON Serializers/De-Serializers

1. Create a new class Quote.java under src/main folder in io.confluent.developer.springccloud package. 

Copy the below code to Quote.java

```java
package io.confluent.developer.springccloud;

import com.fasterxml.jackson.annotation.JsonProperty;

public class Quote {
    @JsonProperty
    public String quote;

    public Quote(String quote) {
        this.quote = quote;
    }
}
```

2. Update pom.xml file to add JSON (De)Serializer related dependency and repository

In pom.xml at line 50, add a new dependency
```
<dependency>
    <groupId>io.confluent</groupId>
    <artifactId>kafka-json-serializer</artifactId>
    <version>7.5.1</version>
</dependency>
```

In pom.xml at line 110, add Confluent maven repository
```
<repositories>
    <repository>
        <id>confluent</id>
        <url>https://packages.confluent.io/maven/</url>
    </repository>
</repositories>
```

The line numbers listed above will only be applicable if you used the provided files without modifications. If you made any changes outside of these instructions, the numbers may not be aligned. Please make the above changes appropriately in the relevant sections.

3. Update the application.properties file to use JSON (De)Serializers

Update spring.kafka.producer.value-serializer at line 10 from org.apache.kafka.common.serialization.StringSerializer to io.confluent.kafka.serializers.KafkaJsonSerializer

Update spring.kafka.consumer.value-deserializer from org.apache.kafka.common.serialization.LongDeserializer to io.confluent.kafka.serializers.KafkaJsonDeserializer


4. Create a new class JsonProducer.java under src/main folder in io.confluent.developer.springccloud package. 

Copy the below code to JsonProducer.java

```java
package io.confluent.developer.springccloud;

import java.time.Duration;
import java.util.stream.Stream;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.annotation.Bean;
import org.springframework.context.event.EventListener;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import com.github.javafaker.Faker;

import lombok.RequiredArgsConstructor;
import reactor.core.publisher.Flux;

@RequiredArgsConstructor
@Component
public class JsonProducer {

    private final KafkaTemplate<Integer, Quote> template;

    Faker faker;

    @Bean
	NewTopic quotesJson() {
		return TopicBuilder.name("quotes-json").partitions(6).replicas(3).build();
	}

    // @EventListener(ApplicationStartedEvent.class)
    public void generate() {

        faker = Faker.instance();
        final Flux<Long> interval = Flux.interval(Duration.ofMillis(1_000));

        final Flux<String> quotes = Flux.fromStream(Stream.generate(() -> faker.hobbit().quote()));

        Flux.zip(interval, quotes)
                .map(it -> {
                    System.out.println("Sending message: " + it.getT2());
                    return template.send("quotes-json", faker.random().nextInt(42), new Quote(it.getT2()));
                })
                .blockLast();
    }
}
```

5. . Create a new class JsonConsumer.java under src/main folder in io.confluent.developer.springccloud package. 

Copy the below code to JsonConsumer.java

```java
package io.confluent.developer.springccloud;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class JsonConsumer {
    @KafkaListener(topics = "quotes-json", groupId = "${spring.kafka.consumer.group-id}")
    public void consume(ConsumerRecord<Integer, Quote> record) {
        System.out.println("received json = " + record.value() + " with key " + record.key());
      }
}
```

6. Being in the root folder in spring-ccloud-maven, run `mvn package`

7. If everything is succesful, you should see messages in the console with the term `received json = `. If you do not see this or see any errors, please check with your instructor/moderator.

8. Once you validate that the messages are successfully consumed you can hit ctrl+c to exit out of the maven process.

## Transactions Support

In this module we add transaction support to the Producer and Consumer code that was created before.

1. Add the following property in application.properties file for the Producer section at line 11.
```spring.kafka.producer.transaction-id-prefix=tx-```

2. Update Producer.java with 2 additional annotations, @EnableTransactionManagement at class level and @Transactional at the method level. The updated class should look like this.

```java
package io.confluent.developer.springccloud;

import java.time.Duration;
import java.util.stream.Stream;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.annotation.Bean;
import org.springframework.context.event.EventListener;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.springframework.transaction.annotation.Transactional;

import com.github.javafaker.Faker;

import lombok.RequiredArgsConstructor;
import reactor.core.publisher.Flux;

@RequiredArgsConstructor
@Component
@EnableTransactionManagement
public class Producer {

    private final KafkaTemplate<Integer, String> template;

    Faker faker;

    @Bean
    NewTopic counts() {
        return TopicBuilder.name("streams-wordcount-output").partitions(6).replicas(3).build();
    }

    @EventListener(ApplicationStartedEvent.class)
    @Transactional
    public void generate() {

        String quote;
        faker = Faker.instance();

        for (int i = 0; i < 5; i++) {
            quote = faker.hobbit().quote();
            System.out.printf("Sending quote %d: %s %n", i, quote);
            template.send("quotes", faker.random().nextInt(42), quote);
        }        
    }
}
```

3. Add the following property in application.properties in the Consumer section
spring.kafka.consumer.properties.isolation.level=read_committed

4. Being in the root folder in spring-ccloud-maven, run `mvn package`

5. If everything is succesful, you should see messages in the console with the term `received = `. If you do not see this or see any errors, please check with your instructor/moderator.

6. Once you validate that the messages are successfully consumed you can hit ctrl+c to exit out of the maven process.