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

7. Open pom.xml. At line 8, update the version value from 3.2.1 to 2.6.5

8. In this step we add logic to produce messages. We will be using a couple of spring project specific and third party libraries to produce random quotes and to send these quotes every second. Create a new java class, Producer.java under src/main folder in io.confluent.developer.springccloud package. The location should look like in the screenshot below 
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

9. 
