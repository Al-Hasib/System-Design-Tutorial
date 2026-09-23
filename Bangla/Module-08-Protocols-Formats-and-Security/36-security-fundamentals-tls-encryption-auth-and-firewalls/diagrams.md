# Diagrams: Security Fundamentals

## ১. TLS: Asymmetric Handshake, তারপর Symmetric Session

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    Note over C,S: Asymmetric crypto (slow, but solves key distribution)
    C->>S: ClientHello
    S-->>C: ServerHello + Certificate (public key)
    C->>C: Verify certificate
    C->>S: Encrypted pre-master secret (using server's public key)
    Note over C,S: Both derive a shared symmetric session key

    Note over C,S: Symmetric crypto (fast) for the actual session
    C->>S: Encrypted HTTP request (AES)
    S-->>C: Encrypted HTTP response (AES)
```
*Asymmetric encryption শুধুমাত্র সংক্ষেপে ব্যবহার করা হয়, আগে থেকে একটি secret share করার প্রয়োজন ছাড়াই নিরাপদে একটি key-তে সম্মত হতে। একবার এটি সম্পন্ন হলে, দ্রুত symmetric algorithm (AES) বাকি সবকিছু encrypt করে।*

## ২. একটি Request-এ Authentication বনাম Authorization

```mermaid
flowchart TD
    A[Request arrives\nwith credentials/token] --> B{Authenticated?\nWho are you?}
    B -->|No / invalid| C[401 Unauthorized]
    B -->|Yes| D{Authorized?\nAllowed to do this?}
    D -->|No| E[403 Forbidden]
    D -->|Yes| F[Process request]
```
*Authentication সবসময় প্রথমে ঘটে — আপনি যাকে identify করেননি তার জন্য permission চেক করতে পারবেন না। 401 মানে identity ব্যর্থ হয়েছে; 403 মানে identity সফল হয়েছে কিন্তু permission হয়নি।*

## ৩. একটি Request-এর পথ জুড়ে Defense in Depth

```mermaid
flowchart LR
    Client[Client] -->|HTTPS/TLS| WAF[WAF\nblocks known attack patterns]
    WAF --> GW[API Gateway\nAuthenticates JWT, checks authorization]
    GW -->|mTLS| SVC[Internal Service\nverifies caller's certificate]
    SVC -->|Restricted by Security Group| DB[(Database\nEncrypted at rest, passwords hashed)]
```
*প্রতিটি স্তর ধরে নেয় যে এর আগেরগুলো ব্যর্থ হতে পারে: WAF কিছু মিস করলেও TLS transit-এ data রক্ষা করে, TLS ইতিমধ্যে চললেও gateway এখনো authorization চেক করে, mTLS "trusted" internal network-এর ভেতরেও caller যাচাই করে, এবং একটি service compromised হলেও database এখনো network rule এবং encryption দ্বারা সুরক্ষিত থাকে।*
