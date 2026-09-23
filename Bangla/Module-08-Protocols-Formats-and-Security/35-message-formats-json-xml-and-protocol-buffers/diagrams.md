# ডায়াগ্রাম: Message Formats — JSON, XML & Protocol Buffers

## ১. তিনটি ভিন্ন Format-এ একই Message

```mermaid
flowchart TB
    subgraph JSON["JSON (text, ~46 bytes)"]
        J["{"id": 123, "name": "Alice"}"]
    end
    subgraph XML["XML (text, ~64 bytes)"]
        X["<user><id>123</id><name>Alice</name></user>"]
    end
    subgraph PB["Protocol Buffers (binary, ~10 bytes)"]
        P["field 1: varint 123, field 2: len-prefixed 'Alice'"]
    end
```

*একই logical data, তিনটি ভিন্ন encoding — JSON এবং XML প্রতিটি message-এ field name/tag text হিসেবে পুনরাবৃত্তি করে; Protocol Buffers একটি shared schema-তে একবার সংজ্ঞায়িত compact numeric field tag দিয়ে name প্রতিস্থাপন করে, যার ফলে অনেক ছোট payload তৈরি হয়।*

## ২. Serialization / Deserialization প্রবাহ

```mermaid
sequenceDiagram
    participant Sender as Sender Service
    participant Wire as Network (bytes)
    participant Receiver as Receiver Service

    Sender->>Sender: Build in-memory object
    Sender->>Sender: Serialize (using agreed format/schema)
    Sender->>Wire: Send byte sequence
    Wire->>Receiver: Deliver byte sequence
    Receiver->>Receiver: Deserialize (using same format/schema)
    Receiver->>Receiver: Use in-memory object
```

*উভয় পক্ষকেই আগে থেকেই সঠিক serialization format নিয়ে সম্মত হতে হবে — এই সম্মতিটাই "message format" বলতে বোঝায়, তা JSON, XML, নাকি একটি shared `.proto` schema হোক না কেন।*

## ৩. Protobuf Schema Evolution: নিরাপদ বনাম অনিরাপদ পরিবর্তন

```mermaid
flowchart LR
    V1["v1 schema\nfield 1: id\nfield 2: name"] -->|"Add field 3: email\n(SAFE — old code ignores it)"| V2["v2 schema\nfield 1: id\nfield 2: name\nfield 3: email"]
    V1 -->|"Reuse field 2 for a different meaning\n(UNSAFE — breaks old/new compatibility)"| Bad["Broken schema"]
```

*একটি নতুন numbered field যোগ করা backward compatible — পুরনো consumer-রা কেবল একটি অচেনা tag এড়িয়ে যায়। একটি বিদ্যমান field number-কে নতুন কিছু বোঝাতে পুনর্বরাদ্দ করলে পুরনো schema চালানো প্রতিটি service ভেঙে যায়।*
</content>
