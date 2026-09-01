# Diagrams: Consistent Hashing

Mermaid does not support true circular/ring layouts, so the ring order is represented with a labeled graph. Node order in each diagram follows the comment "ring order (clockwise)" — read the graph in that sequence to picture the ring wrapping back to the first node.

## 1. The Hash Ring: Nodes and Keys

Caption: Keys are placed on the ring by hash value and owned by the first node found going clockwise.

```mermaid
%% Ring order (clockwise): Node A -> Node B -> Node C -> Node D -> (wraps back to Node A)
graph LR
    K1["Key: user:123<br/>hash = 15"] -.owned by.-> A["Node A<br/>hash = 40"]
    K2["Key: user:456<br/>hash = 55"] -.owned by.-> B["Node B<br/>hash = 90"]
    K3["Key: order:789<br/>hash = 95"] -.owned by.-> C["Node C<br/>hash = 150"]
    K4["Key: order:321<br/>hash = 200"] -.owned by.-> D["Node D<br/>hash = 210"]
    K5["Key: cart:999<br/>hash = 230"] -.owned by.-> A

    A -->|clockwise| B -->|clockwise| C -->|clockwise| D -->|"clockwise (wraps to A)"| A

    style A fill:#4C6EF5,color:#fff
    style B fill:#4C6EF5,color:#fff
    style C fill:#4C6EF5,color:#fff
    style D fill:#4C6EF5,color:#fff
```

## 2. Virtual Nodes Mapping

Caption: Each physical node is hashed multiple times, scattering its virtual points around the ring for even load distribution.

```mermaid
%% Ring order (clockwise): A1 -> B1 -> A2 -> C1 -> B2 -> A3 -> C2 -> B3 -> (wraps back to A1)
graph LR
    subgraph Physical_Node_A["Physical Node A"]
        A1["vnode A#1"]
        A2["vnode A#2"]
        A3["vnode A#3"]
    end
    subgraph Physical_Node_B["Physical Node B"]
        B1["vnode B#1"]
        B2["vnode B#2"]
        B3["vnode B#3"]
    end
    subgraph Physical_Node_C["Physical Node C"]
        C1["vnode C#1"]
        C2["vnode C#2"]
    end

    A1 -->|clockwise| B1 -->|clockwise| A2 -->|clockwise| C1 -->|clockwise| B2 -->|clockwise| A3 -->|clockwise| C2 -->|clockwise| B3 -->|"clockwise (wraps to A1)"| A1

    style A1 fill:#4C6EF5,color:#fff
    style A2 fill:#4C6EF5,color:#fff
    style A3 fill:#4C6EF5,color:#fff
    style B1 fill:#12B886,color:#fff
    style B2 fill:#12B886,color:#fff
    style B3 fill:#12B886,color:#fff
    style C1 fill:#F59F00,color:#000
    style C2 fill:#F59F00,color:#000
```

## 3. Node Addition / Removal: Key Remapping

Caption: Adding Node E only remaps the arc it inherits from Node C; every other node's keys stay put.

```mermaid
%% Ring order (clockwise) BEFORE: Node A -> Node B -> Node C -> (wraps back to Node A)
%% Ring order (clockwise) AFTER:  Node A -> Node B -> Node E -> Node C -> (wraps back to Node A)
graph TB
    subgraph Before["Before: Node E added"]
        direction LR
        A1["Node A<br/>owns arc (C..A]"] -->|clockwise| B1["Node B<br/>owns arc (A..B]"]
        B1 -->|clockwise| C1["Node C<br/>owns arc (B..C]"]
        C1 -->|"clockwise (wraps to A)"| A1
    end

    subgraph After["After: Node E inserted between B and C"]
        direction LR
        A2["Node A<br/>owns arc (C..A]<br/>UNCHANGED"] -->|clockwise| B2["Node B<br/>owns arc (A..B]<br/>UNCHANGED"]
        B2 -->|clockwise| E2["Node E (new)<br/>owns arc (B..E]<br/>keys MOVED from C"]
        E2 -->|clockwise| C2["Node C<br/>owns arc (E..C]<br/>SHRUNK, rest UNCHANGED"]
        C2 -->|"clockwise (wraps to A)"| A2
    end

    Before -.node join / K-over-N keys move.-> After

    style E2 fill:#E64980,color:#fff
    style A2 fill:#4C6EF5,color:#fff
    style B2 fill:#4C6EF5,color:#fff
    style C2 fill:#12B886,color:#fff
```
