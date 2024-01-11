This purpose of this repo is to demostrate a spring boot application interacting with kafka running as part of Confluent Cloud. This example is developed using the Spring Kafka course, https://developer.confluent.io/courses/spring/apache-kafka-intro/

## Table of Contents

- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
- [Instructions](#instructions)

## Getting Started

### Prerequisites

Ensure you have the following installed on your development machine:

- [Java Development Kit (JDK) 17](https://openjdk.java.net/)
- [Apache Maven](https://maven.apache.org/)

## Instructions

1. Open https://start.spring.io/

2. Choose options based on the screenshot listed here ![image](./images/project_settings.png). We recommend to use the values mentioned in the screenshot as it makes it easy to follow the rest of the instructions.

3. Add dependencies based on the screenshot listed here ![image](./images/dependencies.png)

4. Click the Generate button to download the package(zip file) to a folder of your choice. If you used the values shown in the screenshot, the generated filename will be spring-ccloud-maven.zip.  

5. unzip the package to extract the file contents to a folder, spring-ccloud-maven

6. `cd spring-ccloud-maven`

7. In this step we add logic to produce messages. We will be using a couple of spring project specific and third party libraries to produce random quotes and to send these quotes every second. Create a new java class, Producer.java under src/main folder in io.confluent.developer.springccloud package. The location should look like in the screenshot below 
![image](./images/producer_location.png)

Copy the below code to Producer.java

```java
package io.confluent.developer.springccloud;

import java.time.Duration;
import java.util.stream.Stream;

import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.event.EventListener;
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

	@EventListener(ApplicationStartedEvent.class)
	public void generate() {

		faker = Faker.instance();
		final Flux<Long> interval = Flux.interval(Duration.ofMillis(1_000));

		final Flux<String> quotes = Flux.fromStream(Stream.generate(() -> faker.hobbit().quote()));

		Flux.zip(interval, quotes)
				.map(it -> template.send("topic_name", faker.random().nextInt(42), it.getT2())).blockLast();
	}
}
```

8. Open application.properties under src/main/resources. Add the below content to it.
```
# Kafka
spring.kafka.properties.sasl.mechanism=PLAIN
spring.kafka.bootstrap-servers=<TOBEFILLED>
spring.kafka.properties.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<TOBEFILLED>' password='<TOBEFILLED>';
spring.kafka.properties.security.protocol=SASL_SSL

spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.IntegerSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.client-id=spring-boot-producer

spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.LongDeserializer
```
The values for <TOBEFILLED> will be provided by the workshop instructor/moderator

9. Being in the root folder in spring-ccloud-maven, run `mvn package`

10. If you do not see any errors in the console, then random quotes are successfully published to Kafka cluster.

11. Messages should appear in the relevant topic. Open CClould UI, go to the relevant cluster and click on the corresponding topic.  - Need to check with Kishore if they can access UI.
