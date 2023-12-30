---
title: Observability with Grafana Loki and Prometheus with Spring Boot
category: Engineering
---

Observability is a very large topic and it is not something we can write in a single post.
Observability provides a means to gain insights into the performance, health, and behavior
of their applications. It allows for easy debugging, like finding a needle in a haystck. 
At Purple Engineering, Spring Boot is one of the frameworks we use with our Java solutions.
In this blog post, we will explore the concept of observability, its significance in modern 
software architecture, and how Spring Boot can be leveraged to achieve comprehensive observability.

This is one of the essential blocks because some of our applications are not deployed on the cloud 
but on premise infrastructures of our clients. Due to this restrictions we have limited visibility 
into our systems. It is very important for us to understand, gain insights and investigate issues as 
when they happen when we do have access to these systems. The right observability stack allows us to 
do this with ease.

### What is Observability?

Observability refers to the ability to understand the internal state of a system by examining 
its outputs, making it a critical aspect of maintaining and troubleshooting applications. 
Unlike traditional monitoring, observability emphasizes the need for deeper insights, including
detailed logs, metrics, and traces.

Why does Observability matter? Peter Bourgon explained that into details [here](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html)

If observability is done right, there are immense benefits around prerformance optimization and enhanced
 collaboration between development and operations teams. In our case, especially for some of our enterprise
  clients, we do not have real time access to the infrastructe as we focused on development. 


### Observability with Spring Boot

Traces, metrics and logs combined are what helps us achieve perfect observability. Spring boot allows us to easily
 capture these. The Observability stack we would want to use to capture these are 

**1. Logs with [Loki](https://grafana.com/oss/loki/)** <br/>
**2. Metrics with [Prometheus](https://prometheus.io/)**<br/>
**3. Traces with [Tempo](https://grafana.com/oss/tempo/)**<br/>


#### 1. Logs with Loki.


By default, if you use the “Starters”, Logback is used for logging. Appropriate Logback routing is also 
included to ensure that dependent libraries that use Java Util Logging, Commons Logging, Log4J, or SLF4J 
all work correctly. This allows for clean and flexible way to configure logging with ease.

To use Loki as a destination for our logs, we basically need to add a Loki appender. In Spring Boot, an 
appender is simply a destination for log events. 

This can be done in muplitple ways, a simple `logback.xml` file in the default resource location could also suffice.

A stripped down version of a sample configutation will look like this:

```yaml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <include resource="org/springframework/boot/logging/logback/base.xml" />
  <springProperty scope="context" name="appName" source="spring.application.name"/>
  <springProperty scope="context" name="lokiHost" source="loki.host"/>
  <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
      <http>
          <url>${lokiHost}</url>
      </http>
      <format>
          <label>
              <pattern>app=${appName},host=${HOSTNAME},traceID=%X{traceId:-NONE},level=%level</pattern>
          </label>
          <message>
              <pattern>${FILE_LOG_PATTERN}</pattern>
          </message>
          <sortByTime>true</sortByTime>
      </format>
  </appender>
  <root level="INFO">
      <appender-ref ref="LOKI"/>
  </root>
</configuration>
```

Notice as part of the log pattern, we pick the `traceID` when it is available. This allows us to see the correlation between 
a log event and a trace in Tempo.

We also need to ensure all dependecies for logback and the Loki appender are added to the project. 
A maven configuration will look like this:

```yaml
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>${logback.version}</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-core</artifactId>
    <version>${logback.version}</version>
</dependency>
<dependency>
    <groupId>com.github.loki4j</groupId>
    <artifactId>loki-logback-appender</artifactId>
    <version>${loki.appender.version}</version>
</dependency>
```

We can proceed and log events in our Java code and then confirm visibilit in Loki.


#### 2. Metrics with Prometheus.

[Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html) provides built-in support
for exposing application metrics. However, our focus is just promethues so we need to enable just the actuator 
endpoint for Prometheus.

```properties
management.endpoints.web.exposure.include=prometheus
management.prometheus.metrics.export.enabled=true
management.metrics.distribution.percentiles-histogram.http.server.requests=true
```

Of course we need the depencies for this to work.

```yaml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Our application endpoint, `http://localhost:${server.port}/actuator/prometheus` will show the promethues metrics. 
It is as simple as that. Spring Boot does all the rest. If you want to know the full list of metrics we can possibly
get from the actuator endpoint as well as adding your own custom metrics
check out [this](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/production-ready-metrics.html).



#### 3. Traces with Tempo.

For traces, or end-to-end tracing of requests across services. This would require some work in the code to generate
spans for each service/component the request journeys through.
Spring Boot allows us do this with annotations, as well as other means of creating Spans and Traces. 

With support for [Observability in Spring Boot 3](https://spring.io/blog/2022/10/12/observability-with-spring-boot-3/), 
this is much easier.

> For any observation to happen, you need to register ObservationHandler objects through an ObservationRegistry. 
> An ObservationHandler reacts only to supported implementations of an Observation.Context and can create, for example,
 timers, spans, and logs by reacting to the lifecycle events of an observation.

Once we are able to record traces, we need to let our application know where the Otel collector is so as to send the Spans. 
This can be done by adding:

```properties
management.tracing.sampling.probability=1.0
management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
```

You can find more details about Otel collector and configuration using Spring Boot [here](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/html/application-properties.html)


### Conclusion 

Observability is a fundamental aspect of maintaining resilient and high-performance Spring Boot applications. By embracing logging, metrics, and distributed tracing, developers can gain a holistic view of their systems, leading to improved troubleshooting, enhanced performance, and ultimately, a better user experience. As the software landscape continues to evolve, incorporating observability practices becomes essential for building and maintaining robust applications. Hopefully you find our approach of Observability useful.


### References

[https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html)


[https://docs.spring.io/spring-boot/docs/2.1.13.RELEASE/reference/html/boot-features-logging.html](https://docs.spring.io/spring-boot/docs/2.1.13.RELEASE/reference/html/boot-features-logging.html)