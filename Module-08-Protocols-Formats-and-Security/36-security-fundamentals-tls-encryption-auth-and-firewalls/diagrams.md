# Diagrams: Security Fundamentals

## 1. TLS: Asymmetric Handshake, Then Symmetric Session

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
*Asymmetric encryption is used only briefly, to safely agree on a key without needing to share a secret in advance. Once that's done, the fast symmetric algorithm (AES) encrypts everything else.*

## 2. Authentication vs. Authorization in One Request

```mermaid
flowchart TD
    A[Request arrives\nwith credentials/token] --> B{Authenticated?\nWho are you?}
    B -->|No / invalid| C[401 Unauthorized]
    B -->|Yes| D{Authorized?\nAllowed to do this?}
    D -->|No| E[403 Forbidden]
    D -->|Yes| F[Process request]
```
*Authentication always happens first — you can't check permissions for someone you haven't identified. 401 means identity failed; 403 means identity succeeded but permission didn't.*

## 3. Defense in Depth Across a Request's Path

```mermaid
flowchart LR
    Client[Client] -->|HTTPS/TLS| WAF[WAF\nblocks known attack patterns]
    WAF --> GW[API Gateway\nAuthenticates JWT, checks authorization]
    GW -->|mTLS| SVC[Internal Service\nverifies caller's certificate]
    SVC -->|Restricted by Security Group| DB[(Database\nEncrypted at rest, passwords hashed)]
```
*Each layer assumes the ones before it might fail: TLS protects data in transit even if the WAF misses something, the gateway still checks authorization even though TLS already ran, mTLS still verifies the caller even inside the "trusted" internal network, and the database is still protected by network rules and encryption even if a service is compromised.*
