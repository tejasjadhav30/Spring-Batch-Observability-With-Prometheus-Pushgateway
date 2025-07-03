# Spring-Batch-Observability-With-Prometheus-Pushgateway

This project demonstrates how to monitor **Spring Batch jobs** using **Micrometer**, **Prometheus PushGateway**, and **Prometheus**. The metrics are pushed to PushGateway from the batch job, and Prometheus scrapes these metrics at intervals.

---

##  Tech Stack

* **Spring Boot**
* **Spring Batch**
* **Micrometer**
* **Prometheus**
* **Prometheus PushGateway**
* **Grafana** (Optional for visualization)

---

##  Configuration

### 1. `application.yml`

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
        pushgateway:
          enabled: true
          base-url: http://localhost:9091  # PushGateway URL
          job: batch-metrics-job
          push-rate: 5s                   # Rate at which metrics are pushed

```

---

##  Implementation Overview

###  PushGateway Bean

Registers a `PushGateway` bean based on config properties and links it with the Prometheus registry.

```java
@ConditionalOnBean(PrometheusMeterRegistry.class)
@ConditionalOnMissingBean
@ConditionalOnProperty(
    prefix = "management.prometheus.metrics.export.pushgateway",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = false)
@Bean
PushGateway pushGateway(
    PrometheusProperties prometheusProperties,
    PrometheusRegistry prometheusRegistry,
    @Value("${spring.batch.job.name}") String batchJobName)
    throws MalformedURLException {
  URL url = URI.create(prometheusProperties.getPushgateway().getBaseUrl()).toURL();
  PushGateway.Builder builder =
      PushGateway.builder()
          .address(
              prometheusProperties
                  .getPushgateway()
                  .getBaseUrl()
                  .substring(url.getProtocol().length() + 3))
          .scheme(Scheme.fromString(url.getProtocol()))
          .job(
              Optional.ofNullable(prometheusProperties.getPushgateway())
                  .map(PrometheusProperties.Pushgateway::getJob)
                  .orElse(batchJobName));
  prometheusProperties.getPushgateway().getGroupingKey().forEach(builder::groupingKey);
  return builder.registry(prometheusRegistry).build();
}

```

### Push Metrics on Fixed Interval

Uses a simple async scheduler to push metrics periodically.

```java
@Bean
SimpleAsyncTaskScheduler taskSchedulerPushGateway(...) {
    ...
    simpleAsyncTaskScheduler.scheduleAtFixedRate(() -> {
        try {
            pushGateway.push();
        } catch (IOException e) {
            throw new RuntimeException("Failed to push metrics", e);
        }
    }, prometheusProperties.getPushgateway().getPushRate());
}
```

###  Push Metrics on Shutdown

Ensures that the final state of metrics is pushed on app shutdown.

```java
@Bean
ApplicationListener<ContextClosedEvent> shutdownOperationForBatch(...) {
    return event -> {
        try {
            pushGateway.push();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    };
}
```

---

##  Prometheus Configuration

###  Run Prometheus & PushGateway using Docker

Use the following Docker commands to set up Prometheus and PushGateway:

```bash
# Start Prometheus
docker run -d \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  --name prometheus \
  prom/prometheus

# Start PushGateway
docker run -d \
  -p 9091:9091 \
  --name pushgateway \
  prom/pushgateway
```

> Make sure to replace `/path/to/prometheus.yml` with the actual path to your configuration file.

###  Sample `prometheus.yml`

```yaml
scrape_configs:
  - job_name: 'spring-batch'
    honor_labels: true
    static_configs:
      - targets: ['localhost:9091']
```

Prometheus will **pull metrics from PushGateway**, which receives **pushed metrics from Spring Batch**.

---

##  Notes

* `spring.batch.job.name` is injected to label the job in Prometheus.
* Metrics are pushed every `5s` (configurable).
* Suitable for **short-lived batch jobs** or **intermittent jobs** where standard Prometheus pull model fails.

---

##  Grafana (Optional)

* Import the Prometheus data source.


---
## Disadvantages of Using PushGateway
While Prometheus PushGateway is useful for exposing metrics from short-lived batch jobs, it comes with some limitations:

No Retention or Expiry of Metrics
PushGateway does not automatically delete old or stale metrics. Once a metric is pushed, it stays in memory indefinitely until:

Manually deleted by calling the /metrics/job/<job_name>/... DELETE endpoint.

The PushGateway server is restarted.


## 👨‍💼 Author

**Tejas Jadhav**
Intern at **Fyndna Pvt. Ltd.**


---


