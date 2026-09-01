# Diagrams: Message Formats — JSON, XML & Protocol Buffers

## 1. The Same Message in Three Formats

```mermaid
flowchart TB
    subgraph JSON["JSON (text, ~46 bytes)"]
        J["{&quot;id&quot;: 123, &quot;name&quot;: &quot;Alice&quot;}"]
    end
    subgraph XML["XML (text, ~64 bytes)"]
        X["&lt;user&gt;&lt;id&gt;123&lt;/id&gt;&lt;name&gt;Alice&lt;/name&gt;&lt;/user&gt;"]
    end
    subgraph PB["Protocol Buffers (binary, ~10 bytes)"]
        P["field 1: varint 123, field 2: len-prefixed 'Alice'"]
    end
```
*Same logical data, three encodings — JSON and XML repeat field names/tags as text on every message; Protocol Buffers replaces names with compact numeric field tags defined once in a shared schema, producing a much smaller payload.*

## 2. Serialization / Deserialization Flow

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
*Both sides must agree on the exact serialization format ahead of time — that agreement is what "message format" means, whether it's JSON, XML, or a shared `.proto` schema.*

## 3. Protobuf Schema Evolution: Safe vs Unsafe Changes

```mermaid
flowchart LR
    V1["v1 schema\nfield 1: id\nfield 2: name"] -->|"Add field 3: email\n(SAFE — old code ignores it)"| V2["v2 schema\nfield 1: id\nfield 2: name\nfield 3: email"]
    V1 -->|"Reuse field 2 for a different meaning\n(UNSAFE — breaks old/new compatibility)"| Bad["Broken schema"]
```
*Adding a new numbered field is backward compatible — old consumers simply skip a tag they don't recognize. Reassigning an existing field number to mean something new breaks every service still running the old schema.*
