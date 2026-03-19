# 📨 Apache Kafka Best Practices
### Solution Architect Guide — Production-Grade Event-Driven Microservices

> **Audience:** Backend Engineers · Solution Architects · Tech Leads  
> **Stack:** Apache Kafka · Spring Boot 3.x · AWS MSK · Microservices  
> **Version:** 1.0 · March 2026

---

## 📋 Table of Contents

| # | Topic | Priority |
|---|---|---|
| 1 | [Core Concepts & Architecture](#1-core-concepts--architecture) | 🔴 P1 |
| 2 | [Topic Design & Naming](#2-topic-design--naming) | 🔴 P1 |
| 3 | [Producer Best Practices](#3-producer-best-practices) | 🔴 P1 |
| 4 | [Consumer Best Practices](#4-consumer-best-practices) | 🔴 P1 |
| 5 | [Partitioning Strategy](#5-partitioning-strategy) | 🔴 P1 |
| 6 | [Message Schema & Avro](#6-message-schema--avro) | 🔴 P1 |
| 7 | [Error Handling & Dead Letter Queue](#7-error-handling--dead-letter-queue) | 🟡 P2 |
| 8 | [Offset Management](#8-offset-management) | 🟡 P2 |
| 9 | [Security](#9-security) | 🟡 P2 |
| 10 | [Performance & Tuning](#10-performance--tuning) | 🟡 P2 |
| 11 | [Monitoring & Observability](#11-monitoring--observability) | 🟡 P2 |
| 12 | [Kafka on AWS MSK](#12-kafka-on-aws-msk) | 🟢 P3 |
| 13 | [Spring Boot Integration](#13-spring-boot-integration) | 🟢 P3 |

---

## 1. Core Concepts & Architecture

### Kafka Architecture Overview

```mermaid
flowchart TB
    subgraph Producers["📤 Producers"]
        P1[Order Service]
        P2[Payment Service]
        P3[Inventory Service]
    end

    subgraph Kafka["🟠 Kafka Cluster"]
        subgraph Broker1["Broker 1"]
            T1P0[orders P0]
            T1P1[orders P1]
        end
        subgraph Broker2["Broker 2"]
            T1P2[orders P2]
            T2P0[payments P0]
        end
        subgraph Broker3["Broker 3"]
            T2P1[payments P1]
        end
    end

    subgraph Consumers["📥 Consumer Groups"]
        CG1[Notification Service]
        CG2[Analytics Service]
        CG3[Audit Service]
    end

    P1 --> T1P0
    P1 --> T1P1
    P2 --> T2P0
    P3 --> T1P2
    T1P0 --> CG1
    T1P1 --> CG2
    T1P2 --> CG3
    T2P0 --> CG1
    T2P1 --> CG2

    style Producers fill:#E1F5EE,color:#04342C
    style Kafka fill:#FFF8EC,color:#412402
    style Consumers fill:#E6F1FB,color:#042C53
```

### Partition and Consumer Group Layout

```mermaid
flowchart LR
    subgraph Topic["📁 Topic: orders - 3 Partitions"]
        P0["Partition 0\noffset 0,1,2"]
        P1["Partition 1\noffset 0,1"]
        P2["Partition 2\noffset 0,1,2,3"]
    end

    subgraph CG["Consumer Group: notification-grp"]
        C1[Consumer 1]
        C2[Consumer 2]
        C3[Consumer 3]
    end

    P0 --> C1
    P1 --> C2
    P2 --> C3

    style Topic fill:#FFF8EC,color:#412402
    style CG fill:#E6F1FB,color:#042C53
```

### Golden Rules

```
1 partition   → consumed by exactly 1 consumer in a group
1 consumer    → can consume from multiple partitions
consumers > partitions → extra consumers sit IDLE
partitions    = max parallelism for a consumer group
same key      → always same partition → ordered per key
different partitions → NOT ordered across each other
```

---

## 2. Topic Design & Naming

### Naming Convention

```
Format:   {domain}.{entity}.{event-type}

Examples:
  orders.order.created
  payments.payment.failed
  inventory.stock.updated
  users.user.registered

With environment prefix:
  prod.orders.order.created
  staging.orders.order.created
  dev.orders.order.created
```

### Topic Design Decision Flow

```mermaid
flowchart TD
    A[What is the event?] --> B{Who consumes it?}
    B -- Single team --> C[One topic per event type]
    B -- Multiple teams --> D[Use Avro and Schema Registry]
    C --> E[orders.order.created]
    D --> F[Schema Registry validates contract]
    F --> G[Consumers evolve independently]

    style A fill:#378ADD,color:#fff
    style C fill:#1D9E75,color:#fff
    style D fill:#7F77DD,color:#fff
```

### Topic Configuration Reference

```yaml
orders.order.created:
  partitions: 12
  replication-factor: 3
  retention.ms: 604800000         # 7 days
  cleanup.policy: delete
  min.insync.replicas: 2
  compression.type: lz4
```

### Common Topic Design Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| One topic for everything | Consumers get irrelevant events | Separate topic per event type |
| Too few partitions (1-2) | Cannot scale consumers | Start with 12 partitions minimum |
| Replication factor = 1 | Broker failure = data loss | Always use 3 in production |
| Short retention (1 hour) | Consumer outage = missed events | Use 7 days minimum |
| Topic names with spaces | Breaks tooling | Use dots or hyphens only |

---

## 3. Producer Best Practices

### Producer Send Flow

```mermaid
sequenceDiagram
    participant App as Spring Boot App
    participant Prod as Kafka Producer
    participant Buf as Record Buffer
    participant Broker as Kafka Broker

    App->>Prod: send(ProducerRecord)
    Prod->>Buf: batch record
    Note over Buf: Wait for batch.size OR linger.ms
    Buf->>Broker: send batch
    Broker-->>Prod: RecordMetadata
    Prod-->>App: Future complete
```

### Producer Acknowledgement Modes

```mermaid
flowchart TD
    subgraph acks0["acks=0 — Fire and Forget"]
        A1[Producer sends message]
        A2[Does NOT wait for ack]
        A3[Fastest — messages can be lost]
    end
    subgraph acks1["acks=1 — Leader Only"]
        B1[Producer sends message]
        B2[Waits for leader ack only]
        B3[Fast — leader crash causes data loss]
    end
    subgraph acksAll["acks=all — Recommended"]
        C1[Producer sends message]
        C2[Waits for all ISR acks]
        C3[Slowest — guaranteed no data loss]
    end

    style acks0 fill:#FAECE7,color:#4A1B0C
    style acks1 fill:#FFF8EC,color:#412402
    style acksAll fill:#E1F5EE,color:#04342C
```

### Producer Configuration

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      acks: all
      retries: 3
      retry-backoff-ms: 1000
      batch-size: 32768
      linger-ms: 5
      buffer-memory: 33554432
      compression-type: lz4
      enable-idempotence: true
      max-in-flight-requests-per-connection: 5
      properties:
        schema.registry.url: ${SCHEMA_REGISTRY_URL}
```

### Idempotent Producer — Why It Matters

```
Without idempotence:
  Producer sends → network hiccup → producer retries → DUPLICATE message

With enable.idempotence=true:
  Producer sends → network hiccup → producer retries
  Broker detects duplicate via sequence number → drops it
  Result: exactly-once delivery within a session
```

### Producer Code

```java
@Service
@Slf4j
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    private static final String TOPIC = "orders.order.created";

    public void publishOrderCreated(Order order) {
        OrderEvent event = OrderEvent.builder()
            .orderId(order.getId())
            .customerId(order.getCustomerId())
            .status(order.getStatus())
            .timestamp(Instant.now())
            .build();

        // Always use a meaningful key for ordering
        kafkaTemplate.send(TOPIC, order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish orderId={} error={}",
                        order.getId(), ex.getMessage());
                } else {
                    log.info("Published orderId={} partition={} offset={}",
                        order.getId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

---

## 4. Consumer Best Practices

### Consumer Group Rebalancing

```mermaid
sequenceDiagram
    participant C1 as Consumer 1
    participant C2 as Consumer 2
    participant C3 as Consumer 3 NEW
    participant CG as Group Coordinator

    Note over C1,C2: C1 reads P0 and P1. C2 reads P2 and P3.
    C3->>CG: Join group
    CG->>C1: Stop consuming — rebalance triggered
    CG->>C2: Stop consuming — rebalance triggered
    Note over C1,C2,C3: All consumers paused during rebalance
    CG->>C1: Assigned Partition 0
    CG->>C2: Assigned Partitions 1 and 2
    CG->>C3: Assigned Partition 3
    Note over C1,C2,C3: Consuming resumes with new assignment
```

> **Warning:** Rebalancing pauses ALL consumers in the group.  
> Minimise rebalances by keeping consumers alive and tuning `session.timeout.ms`.

### Consumer Configuration

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
      group-id: order-notification-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false        # Always false — use manual commit
      max-poll-records: 100
      fetch-min-bytes: 1024
      fetch-max-wait-ms: 500
      session-timeout-ms: 30000
      heartbeat-interval-ms: 10000
      max-poll-interval-ms: 300000
      properties:
        schema.registry.url: ${SCHEMA_REGISTRY_URL}
        isolation.level: read_committed
```

### Consumer Code

```java
@Service
@Slf4j
public class OrderEventConsumer {

    @KafkaListener(
        topics = "orders.order.created",
        groupId = "notification-service",
        concurrency = "3"
    )
    public void consumeOrderCreated(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {

        log.info("Consuming orderId={} partition={} offset={}",
            event.getOrderId(), partition, offset);

        try {
            notificationService.sendOrderConfirmation(event);

            // Commit ONLY after successful processing
            acknowledgment.acknowledge();

        } catch (NonRetryableException ex) {
            // Business error — send to DLQ, do not retry
            log.error("Non-retryable error orderId={}", event.getOrderId(), ex);
            deadLetterQueueService.send(event, ex);
            acknowledgment.acknowledge();

        } catch (RetryableException ex) {
            // Transient error — do NOT commit, Spring retries
            log.warn("Retryable error orderId={} will retry", event.getOrderId());
            throw ex;
        }
    }
}
```

### Concurrency vs Partition Count Rule

```
Rule: concurrency must NEVER exceed partition count

Topic has 6 partitions:
  concurrency=3 → 3 threads each reading 2 partitions  OK
  concurrency=6 → 6 threads each reading 1 partition   OK (max parallelism)
  concurrency=8 → 8 threads but 2 sit IDLE             WASTE

In Kubernetes:
  3 pods x concurrency=2 = 6 consumer threads = 6 partitions  IDEAL
```

---

## 5. Partitioning Strategy

### How Partitioning Works

```mermaid
flowchart TD
    A[ProducerRecord with KEY] --> B{Key present?}
    B -- Yes --> C[hash of key mod num partitions]
    B -- No --> D[Round robin across partitions]
    C --> E[Same key always same partition]
    D --> F[Balanced load but no ordering]
    E --> G[Ordered per key guaranteed]
    F --> H[No order guarantee]

    style E fill:#1D9E75,color:#fff
    style G fill:#1D9E75,color:#fff
    style H fill:#EF9F27,color:#fff
```

### Key Selection Strategy

```mermaid
flowchart LR
    subgraph GoodKeys["Good Keys"]
        K1[orderId — all events for one order stay ordered]
        K2[customerId — all customer events stay ordered]
        K3[productId — all product events stay ordered]
    end
    subgraph BadKeys["Bad Keys — Avoid"]
        K4[random UUID — no ordering benefit]
        K5[timestamp — hot partition at peak time]
        K6[null — round robin no ordering]
    end

    style GoodKeys fill:#E1F5EE,color:#04342C
    style BadKeys fill:#FAECE7,color:#4A1B0C
```

### Hot Partition Problem and Fix

```mermaid
flowchart TD
    subgraph Problem["Hot Partition Problem"]
        HP1[Key = userId 1001 — 90% of traffic]
        HP2[Partition 0 OVERLOADED]
        HP3[Partition 1 idle]
        HP4[Partition 2 idle]
        HP1 --> HP2
    end
    subgraph Solution["Solution — Compound Key"]
        HS1[Key = userId plus random suffix]
        HS2[Partition 0 balanced]
        HS3[Partition 1 balanced]
        HS4[Partition 2 balanced]
        HS1 --> HS2
        HS1 --> HS3
        HS1 --> HS4
    end

    style Problem fill:#FAECE7,color:#4A1B0C
    style Solution fill:#E1F5EE,color:#04342C
```

### Partition Count Guidelines

```
Low throughput  under 1000 msg/s      →  6 partitions
Medium          1000 to 10000 msg/s   →  12 partitions
High            over 10000 msg/s      →  24 to 48 partitions

Formula:
  target_partitions = max(
    target_throughput / throughput_per_partition,
    target_consumer_count
  )

  throughput_per_partition ≈ 10 MB/s write, 30 MB/s read

Warning: Partitions can INCREASE but NEVER decrease. Plan ahead!
```

---

## 6. Message Schema & Avro

### Why Schema Registry

```mermaid
flowchart LR
    subgraph Without["Without Schema Registry"]
        A1[Producer sends plain JSON]
        A2[Consumer breaks when field added]
        A3[No contract between teams]
        A4[Schema drifts silently over time]
    end
    subgraph With["With Avro and Schema Registry"]
        B1[Producer registers schema]
        B2[Schema Registry validates on publish]
        B3[Consumer evolves independently]
        B4[Only backward compatible changes allowed]
    end

    style Without fill:#FAECE7,color:#4A1B0C
    style With fill:#E1F5EE,color:#04342C
```

### Avro Schema Example

```json
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.company.orders.events",
  "fields": [
    { "name": "orderId",     "type": "long" },
    { "name": "customerId",  "type": "long" },
    { "name": "status",      "type": "string" },
    { "name": "totalAmount", "type": "double" },
    { "name": "currency",    "type": "string", "default": "INR" },
    { "name": "timestamp",   "type": "long", "logicalType": "timestamp-millis" },
    {
      "name": "metadata",
      "type": ["null", { "type": "map", "values": "string" }],
      "default": null,
      "doc": "Optional field — backward compatible"
    }
  ]
}
```

### Schema Evolution Rules

```
SAFE — Backward Compatible (deploy consumer first):
  Add optional field with default value
  Remove field that has a default value

SAFE — Forward Compatible (deploy producer first):
  Add any new field
  Remove an optional field

BREAKING — Never do these:
  Rename a field
  Change field type (int to string)
  Remove a required field without default
  Add a required field without default
```

---

## 7. Error Handling & Dead Letter Queue

### Error Handling Flow

```mermaid
flowchart TD
    A[Message consumed] --> B[Process message]
    B --> C{Success?}
    C -- Yes --> D[Commit offset]
    C -- No --> E{Retryable error?}
    E -- Yes --> F[Retry with exponential backoff]
    F --> G{Max retries reached?}
    G -- No --> B
    G -- Yes --> H[Send to Dead Letter Queue]
    E -- No --> H
    H --> I[Commit offset]
    I --> J[Alert operations team]
    J --> K[Manual inspection and replay]

    style D fill:#1D9E75,color:#fff
    style H fill:#E24B4A,color:#fff
    style K fill:#EF9F27,color:#fff
```

### Dead Letter Queue Topology

```
Main topic:    orders.order.created
Retry topics:  orders.order.created.retry-1   (wait 1s)
               orders.order.created.retry-2   (wait 10s)
               orders.order.created.retry-3   (wait 60s)
DLQ topic:     orders.order.created.dlt       (manual review)

Flow:
  Message fails
    → retry-1 after 1s
    → retry-2 after 10s
    → retry-3 after 60s
    → DLQ — requires manual intervention
```

### Spring Boot DLQ Configuration

```java
@Configuration
public class KafkaErrorHandlerConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> kafkaTemplate) {

        ExponentialBackOffWithMaxRetries backoff =
            new ExponentialBackOffWithMaxRetries(3);
        backoff.setInitialInterval(1_000L);
        backoff.setMultiplier(10.0);
        backoff.setMaxInterval(60_000L);

        DeadLetterPublishingRecoverer recoverer =
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(
                    record.topic() + ".dlt",
                    record.partition()
                ));

        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backoff);

        handler.addNotRetryableExceptions(
            IllegalArgumentException.class,
            ValidationException.class
        );

        return handler;
    }
}
```

### DLQ Consumer for Manual Replay

```java
@Service
@Slf4j
public class DeadLetterQueueConsumer {

    @KafkaListener(
        topics = "orders.order.created.dlt",
        groupId = "dlt-inspector"
    )
    public void consumeDeadLetter(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage,
            @Header(KafkaHeaders.ORIGINAL_TOPIC) String originalTopic,
            @Header(KafkaHeaders.ORIGINAL_OFFSET) long originalOffset) {

        log.error("DLQ message | topic={} offset={} error={}",
            originalTopic, originalOffset, errorMessage);

        dlqRepository.save(DlqEntry.builder()
            .originalTopic(originalTopic)
            .originalOffset(originalOffset)
            .errorMessage(errorMessage)
            .payload(event.toString())
            .receivedAt(Instant.now())
            .build());

        alertService.sendDlqAlert(originalTopic, errorMessage);
    }
}
```

---

## 8. Offset Management

### Auto Commit vs Manual Commit

```mermaid
flowchart TD
    subgraph Auto["Auto Commit — DANGEROUS"]
        AC1[Consumer polls message]
        AC2[Auto-commits after 5 seconds]
        AC3[App crashes before processing]
        AC4[Offset committed but message never processed]
        AC5[Message lost forever]
        AC1 --> AC2 --> AC3 --> AC4 --> AC5
    end
    subgraph Manual["Manual Commit — CORRECT"]
        MC1[Consumer polls message]
        MC2[Process message successfully]
        MC3[Manually commit offset]
        MC4[App crashes before commit]
        MC5[Restart and reprocess from last committed offset]
        MC1 --> MC2 --> MC3
        MC1 --> MC4 --> MC5
    end

    style Auto fill:#FAECE7,color:#4A1B0C
    style Manual fill:#E1F5EE,color:#04342C
```

### Delivery Guarantee Comparison

| Guarantee | How | Duplicates | Data Loss | Use When |
|---|---|---|---|---|
| At-most-once | Commit before processing | Never | Possible | Logging, analytics |
| At-least-once | Commit after processing | Possible | Never | Most business events |
| Exactly-once | Kafka transactions | Never | Never | Financial transactions |

### Idempotent Consumer Pattern

```java
@Service
public class OrderEventConsumer {

    @KafkaListener(topics = "orders.order.created")
    public void consume(OrderEvent event, Acknowledgment ack) {

        // Check if already processed — prevents duplicate side effects
        if (processedEventRepository.existsByEventId(event.getEventId())) {
            log.info("Duplicate event skipped eventId={}", event.getEventId());
            ack.acknowledge();
            return;
        }

        notificationService.send(event);

        processedEventRepository.save(
            ProcessedEvent.of(event.getEventId(), Instant.now())
        );

        ack.acknowledge();
    }
}
```

---

## 9. Security

### Kafka Security Layers

```mermaid
flowchart TD
    A[Client connects] --> B[TLS Encryption in transit]
    B --> C{Authentication method}
    C --> D[SASL SCRAM — username and password]
    C --> E[SASL GSSAPI — Kerberos]
    C --> F[mTLS — certificate based]
    D --> G[ACL Authorisation check]
    E --> G
    F --> G
    G --> H{Has permission?}
    H -- Yes --> I[Topic access granted]
    H -- No --> J[Access denied and logged]

    style B fill:#378ADD,color:#fff
    style G fill:#7F77DD,color:#fff
    style I fill:#1D9E75,color:#fff
    style J fill:#E24B4A,color:#fff
```

### ACL Best Practices

```bash
# Producer — write only to specific topic
kafka-acls.sh --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:order-service \
  --operation Write \
  --topic orders.order.created

# Consumer — read only with specific group
kafka-acls.sh --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:notification-service \
  --operation Read \
  --topic orders.order.created \
  --group notification-service-grp

# Never do this — too permissive
# --allow-principal User:* --operation All --topic '*'
```

### SSL Configuration in Spring Boot

```yaml
spring:
  kafka:
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-512
      sasl.jaas.config: >
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="${KAFKA_USERNAME}"
        password="${KAFKA_PASSWORD}";
      ssl.truststore.location: /etc/kafka/ssl/kafka.truststore.jks
      ssl.truststore.password: ${KAFKA_TRUSTSTORE_PASSWORD}
```

---

## 10. Performance & Tuning

### Producer Tuning — Choose Your Priority

```mermaid
flowchart LR
    subgraph Throughput["Maximise Throughput"]
        T1[linger.ms = 20]
        T2[batch.size = 65536]
        T3[compression.type = lz4]
        T4[acks = 1]
    end
    subgraph Latency["Minimise Latency"]
        L1[linger.ms = 0]
        L2[batch.size = 16384]
        L3[compression.type = none]
        L4[acks = 1]
    end
    subgraph Reliable["Maximise Reliability"]
        R1[acks = all]
        R2[enable.idempotence = true]
        R3[retries = 3]
        R4[min.insync.replicas = 2]
    end

    style Throughput fill:#E1F5EE,color:#04342C
    style Latency fill:#E6F1FB,color:#042C53
    style Reliable fill:#EEEDFE,color:#26215C
```

### Consumer Tuning

```yaml
consumer:
  max-poll-records: 500
  fetch-min-bytes: 102400
  fetch-max-wait-ms: 500
  max-poll-interval-ms: 600000
  session-timeout-ms: 45000
  heartbeat-interval-ms: 15000
```

### Key Metrics and Thresholds

| Metric | Warning | Critical | Action |
|---|---|---|---|
| Consumer lag | > 10,000 | > 100,000 | Add consumers or partitions |
| Producer error rate | > 0.1% | > 1% | Check broker health |
| Broker disk usage | > 70% | > 85% | Reduce retention or add brokers |
| Under-replicated partitions | > 0 | > 5 | Check broker connectivity |
| Request handler idle | < 30% | < 10% | Add brokers |

---

## 11. Monitoring & Observability

### Monitoring Stack

```mermaid
flowchart TB
    subgraph Kafka["Kafka Cluster — JMX Metrics"]
        B1[Broker 1]
        B2[Broker 2]
        B3[Broker 3]
    end
    subgraph Apps["Applications"]
        P[Producer metrics]
        C[Consumer lag metrics]
    end
    subgraph Collection["Collection Layer"]
        JE[JMX Exporter]
        PE[Prometheus]
    end
    subgraph Viz["Visualisation and Alerting"]
        GR[Grafana Dashboards]
        AM[Alert Manager]
        SL[Slack and PagerDuty]
    end

    B1 --> JE
    B2 --> JE
    B3 --> JE
    P --> PE
    C --> PE
    JE --> PE
    PE --> GR
    PE --> AM
    AM --> SL

    style Kafka fill:#FFF8EC,color:#412402
    style Apps fill:#E1F5EE,color:#04342C
    style Viz fill:#E6F1FB,color:#042C53
```

### Consumer Lag Explained

```mermaid
flowchart LR
    subgraph Partition["Topic Partition"]
        M1[offset 0]
        M2[offset 1 — committed]
        M3[offset 2]
        M4[offset 3 — latest]
    end

    LAG["LAG = latest offset minus committed offset\n= 3 minus 1 = 2\nConsumer is 2 messages behind"]

    Partition --> LAG

    style LAG fill:#EF9F27,color:#fff
```

### Critical Alerts

```yaml
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_group_lag > 10000
  for: 5m
  annotations:
    summary: "Consumer lag too high on topic {{ $labels.topic }}"

- alert: KafkaUnderReplicatedPartitions
  expr: kafka_server_replicamanager_underreplicatedpartitions > 0
  for: 1m
  annotations:
    summary: "Under-replicated partitions detected"

- alert: KafkaDLQMessageDetected
  expr: kafka_topic_partitions_messages_in_rate{topic=~".*\\.dlt"} > 0
  for: 1m
  annotations:
    summary: "Messages arriving in Dead Letter Queue"

- alert: KafkaProducerErrorRate
  expr: rate(kafka_producer_record_error_rate[5m]) > 0.01
  for: 2m
  annotations:
    summary: "Producer error rate above 1%"
```

---

## 12. Kafka on AWS MSK

### MSK Architecture

```mermaid
flowchart TB
    subgraph MSK["AWS MSK Cluster — Same VPC as EKS"]
        subgraph AZ1["AZ ap-south-1a"]
            BR1[Broker 1\nkafka.m5.large]
        end
        subgraph AZ2["AZ ap-south-1b"]
            BR2[Broker 2\nkafka.m5.large]
        end
        subgraph AZ3["AZ ap-south-1c"]
            BR3[Broker 3\nkafka.m5.large]
        end
    end
    subgraph EKS["EKS Cluster"]
        PS[Producer Pods]
        CS[Consumer Pods]
    end
    subgraph Infra["Supporting Services"]
        SR[Schema Registry]
        CW[CloudWatch Metrics]
    end

    PS --> BR1
    PS --> BR2
    CS --> BR2
    CS --> BR3
    BR1 --> CW
    BR2 --> CW
    BR3 --> CW
    PS --> SR
    CS --> SR

    style MSK fill:#FFF8EC,color:#412402
    style EKS fill:#E1F5EE,color:#04342C
```

### MSK Setup Checklist

```
Infrastructure:
  3 brokers across 3 AZs minimum
  kafka.m5.large for medium workloads
  MSK in same VPC as EKS
  Private subnets only — no public access
  Security group: allow 9094 TLS from EKS SG only

Storage:
  EBS gp3 — better performance than gp2
  Storage auto-scaling enabled
  Start with 100GB per broker

Security:
  TLS encryption in-transit enabled
  SASL SCRAM authentication enabled
  Unauthenticated access disabled

Reliability:
  replication.factor=3 on all topics
  min.insync.replicas=2 on all topics
  Automatic failover enabled

Monitoring:
  CloudWatch metrics enabled
  Broker log delivery to S3
  Consumer lag monitored via CloudWatch
```

---

## 13. Spring Boot Integration

### Full Kafka Configuration Bean

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        props.put("specific.avro.reader", true);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties()
            .setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        factory.setCommonErrorHandler(errorHandler(kafkaTemplate()));
        return factory;
    }

    @Bean
    public NewTopic orderCreatedTopic() {
        return TopicBuilder.name("orders.order.created")
            .partitions(12)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "604800000")
            .config(TopicConfig.COMPRESSION_TYPE_CONFIG, "lz4")
            .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
            .build();
    }

    @Bean
    public NewTopic orderCreatedDltTopic() {
        return TopicBuilder.name("orders.order.created.dlt")
            .partitions(12)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "2592000000")
            .build();
    }
}
```

### Transactional Outbox Pattern

```mermaid
sequenceDiagram
    participant Svc as Order Service
    participant DB as PostgreSQL
    participant Out as Outbox Table
    participant Rel as Outbox Relay
    participant KF as Kafka Topic

    Svc->>DB: INSERT order — begin transaction
    Svc->>Out: INSERT outbox event — same transaction
    DB-->>Svc: COMMIT — both writes atomic
    Rel->>Out: Poll unpublished events every 1s
    Rel->>KF: Publish event to Kafka
    KF-->>Rel: Ack received
    Rel->>Out: Mark event as published
```

```java
@Transactional
public Order createOrder(CreateOrderRequest request) {

    // Step 1 — save order
    Order order = orderRepository.save(orderMapper.toEntity(request));

    // Step 2 — save outbox event IN SAME TRANSACTION
    outboxRepository.save(OutboxEvent.builder()
        .aggregateId(order.getId().toString())
        .eventType("OrderCreated")
        .payload(objectMapper.writeValueAsString(order))
        .published(false)
        .createdAt(Instant.now())
        .build());

    // Both commit atomically — no ghost events, no missed events
    return order;
}

// Relay publishes from outbox to Kafka every second
@Scheduled(fixedDelay = 1000)
public void relayOutboxEvents() {
    outboxRepository.findByPublishedFalse().forEach(event -> {
        kafkaTemplate.send(topicFor(event.getEventType()),
            event.getAggregateId(), event.getPayload());
        event.setPublished(true);
        outboxRepository.save(event);
    });
}
```

---

## 📊 Complete Checklist

### Topic Design
- [ ] Naming convention `domain.entity.event-type` followed
- [ ] Minimum 12 partitions for production topics
- [ ] Replication factor = 3
- [ ] min.insync.replicas = 2
- [ ] Retention minimum 7 days
- [ ] DLQ topic created per consumer topic
- [ ] Compression enabled with lz4

### Producer
- [ ] acks = all
- [ ] enable.idempotence = true
- [ ] linger.ms and batch.size tuned
- [ ] Meaningful message key set
- [ ] Avro schema registered in Schema Registry
- [ ] Send result callback handled
- [ ] Transactional outbox used for DB and Kafka atomicity

### Consumer
- [ ] enable.auto.commit = false
- [ ] Manual offset commit after successful processing
- [ ] concurrency does not exceed partition count
- [ ] Idempotent consumer with duplicate check
- [ ] DLQ configured with retry backoff
- [ ] Non-retryable exceptions go direct to DLQ
- [ ] max.poll.interval.ms tuned to processing time

### Security
- [ ] TLS encryption in transit
- [ ] SASL authentication enabled
- [ ] ACLs per service with least privilege
- [ ] No wildcard ACLs in production
- [ ] Credentials from Secrets Manager

### Observability
- [ ] Consumer lag monitored and alerted
- [ ] Under-replicated partitions alerted
- [ ] DLQ arrivals alerted immediately
- [ ] Producer error rate alerted
- [ ] Grafana dashboard for all Kafka metrics

### AWS MSK
- [ ] 3 brokers across 3 AZs
- [ ] Same VPC as EKS
- [ ] Private subnets only
- [ ] Storage auto-scaling enabled
- [ ] CloudWatch metrics enabled

---

## 🚦 Priority Summary

| Priority | Practice | Risk if Skipped |
|---|---|---|
| 🔴 P1 | acks=all with min.insync.replicas=2 | Data loss on broker failure |
| 🔴 P1 | enable.auto.commit=false | Messages silently lost on crash |
| 🔴 P1 | replication-factor=3 | Single broker failure loses data |
| 🔴 P1 | Meaningful message keys | No ordering and hot partitions |
| 🔴 P1 | DLQ configured | Failed messages lost with no trace |
| 🔴 P1 | Avro and Schema Registry | Schema drift silently breaks consumers |
| 🟡 P2 | Idempotent consumer | Duplicate processing on rebalance |
| 🟡 P2 | Consumer lag monitoring | Silent backlog builds up unnoticed |
| 🟡 P2 | Retry with exponential backoff | Thundering herd on transient failures |
| 🟡 P2 | Transactional outbox pattern | Ghost events or silently missed events |
| 🟡 P2 | ACLs per service | Any service can read any topic |
| 🟢 P3 | lz4 compression | Higher storage and network costs |
| 🟢 P3 | Partition count tuning | Suboptimal consumer parallelism |
| 🟢 P3 | Multi-stage lag alerts | Slow response to consumer lag issues |

---

*Document version 1.0 · Solution Architecture Team · March 2026*  
*Apache Kafka · Spring Boot 3.x · AWS MSK · Production Grade*
