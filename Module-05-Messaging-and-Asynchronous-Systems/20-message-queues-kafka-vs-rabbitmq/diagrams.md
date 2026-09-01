# Diagrams: Message Queues, Kafka vs RabbitMQ

## 1. Basic Producer-Consumer Flow via a Message Queue

```mermaid
flowchart LR
    P[Producer] -->|1: publish message| B[(Broker / Queue)]
    B -->|2: deliver message| C[Consumer]
    C -->|3: acknowledge| B
```
*A producer publishes a message to the broker and moves on; the consumer processes it independently and acknowledges completion.*

## 2. RabbitMQ-Style Routing with a Consumer Group

```mermaid
flowchart LR
    Producer -->|publish| Exchange{Exchange}
    Exchange -->|route| Q1[(Order Queue)]
    Q1 --> W1[Worker 1]
    Q1 --> W2[Worker 2]
    Q1 --> W3[Worker 3]
```
*RabbitMQ routes messages through an exchange into a queue; multiple workers compete for messages, each message handled by exactly one worker.*

## 3. Kafka Topic, Partitions, and Independent Consumer Groups

```mermaid
flowchart TB
    Producer --> T[Topic: orders]
    subgraph T[Topic: orders]
        P0[Partition 0]
        P1[Partition 1]
        P2[Partition 2]
    end
    P0 --> CG1A[Consumer Group A - instance 1]
    P1 --> CG1B[Consumer Group A - instance 2]
    P2 --> CG1C[Consumer Group A - instance 3]
    P0 --> CG2[Consumer Group B - Analytics]
    P1 --> CG2
    P2 --> CG2
```
*Kafka partitions a topic for parallel consumption within a group, while entirely separate consumer groups can independently replay the same data.*
