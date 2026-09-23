# ডায়াগ্রাম: Message Queues, Kafka vs RabbitMQ

## ১. Message Queue-এর মাধ্যমে মৌলিক Producer-Consumer Flow

```mermaid
flowchart LR
    P[Producer] -->|1: publish message| B[(Broker / Queue)]
    B -->|2: deliver message| C[Consumer]
    C -->|3: acknowledge| B
```
*একজন producer broker-কে একটি message publish করে এগিয়ে যায়; consumer সেটিকে স্বাধীনভাবে প্রসেস করে এবং সম্পন্ন হওয়ার acknowledgment দেয়।*

## ২. Consumer Group-সহ RabbitMQ-স্টাইল Routing

```mermaid
flowchart LR
    Producer -->|publish| Exchange{Exchange}
    Exchange -->|route| Q1[(Order Queue)]
    Q1 --> W1[Worker 1]
    Q1 --> W2[Worker 2]
    Q1 --> W3[Worker 3]
```
*RabbitMQ একটি exchange-এর মাধ্যমে message-কে একটি queue-তে রুট করে; একাধিক worker message-এর জন্য প্রতিযোগিতা করে, প্রতিটি message ঠিক একজন worker দ্বারা সামলানো হয়।*

## ৩. Kafka Topic, Partitions, এবং স্বতন্ত্র Consumer Groups

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
*Kafka একটি group-এর মধ্যে সমান্তরাল consumption-এর জন্য একটি topic-কে partition করে, আর সম্পূর্ণ আলাদা consumer group-রা স্বাধীনভাবে একই ডেটা replay করতে পারে।*
</content>
