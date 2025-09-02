# Commonware Broadcast - C4 Architecture & Flow Documentation

## Table of Contents

1. [Overview](#overview)
2. [C4 Level 1: System Context](#c4-level-1-system-context)
3. [C4 Level 2: Container](#c4-level-2-container)
4. [C4 Level 3: Component](#c4-level-3-component)
5. [C4 Level 4: Code](#c4-level-4-code)
6. [Critical Process Flows](#critical-process-flows)
7. [Data Models](#data-models)

---

## Overview

The `commonware-broadcast` library is a Rust crate that provides functionality to disseminate data over a wide-area network. It implements a distributed broadcasting system with message caching, peer-to-peer communication, and efficient message retrieval.

### Key Features

- **Message Broadcasting**: Disseminate messages to selected or all network peers
- **Message Caching**: LRU cache with configurable size per peer
- **Asynchronous Operations**: Built on futures and async/await
- **Metrics & Telemetry**: Comprehensive monitoring capabilities
- **Flexible Recipients**: Support for targeted, selective, or broadcast messaging

---

## C4 Level 1: System Context

The system context shows how the broadcast system interacts with external actors and systems.

```mermaid
graph TB
    subgraph "Wide Area Network"
        User["Application/User<br/>(System Actor)"]
        Network["P2P Network<br/>(External System)"]
        Metrics["Metrics System<br/>(External System)"]
    end

    Broadcast["Commonware Broadcast<br/>(Software System)"]

    User -->|"Broadcast messages<br/>Subscribe to messages<br/>Retrieve messages"| Broadcast
    Broadcast <-->|"Send/Receive<br/>messages"| Network
    Broadcast -->|"Export telemetry<br/>data"| Metrics

    style Broadcast fill:#f9f,stroke:#333,stroke-width:4px
    style User fill:#bbf,stroke:#333,stroke-width:2px
    style Network fill:#bfb,stroke:#333,stroke-width:2px
    style Metrics fill:#fbf,stroke:#333,stroke-width:2px
```

### External Actors

- **Application/User**: Systems or applications that use the broadcast library
- **P2P Network**: The underlying peer-to-peer network infrastructure
- **Metrics System**: External monitoring and observability systems (e.g., Prometheus)

---

## C4 Level 2: Container

The container diagram shows the high-level technical building blocks within the broadcast system.

```mermaid
graph TB
    subgraph "Commonware Broadcast System"
        Engine["Engine<br/>(Core Runtime)<br/>- Message processing<br/>- Cache management<br/>- Network coordination"]
        Mailbox["Mailbox<br/>(API Interface)<br/>- Async message queue<br/>- Request handling"]
        Cache["Message Cache<br/>(Storage)<br/>- LRU per-peer deques<br/>- Reference counting"]
        Metrics["Metrics Module<br/>(Telemetry)<br/>- Performance counters<br/>- Status tracking"]
    end

    subgraph "External Dependencies"
        P2P["commonware-p2p<br/>(Network Layer)"]
        Crypto["commonware-cryptography<br/>(Security)"]
        Runtime["commonware-runtime<br/>(Execution)"]
        Codec["commonware-codec<br/>(Serialization)"]
    end

    Mailbox --> Engine
    Engine --> Cache
    Engine --> Metrics
    Engine <--> P2P
    Engine --> Crypto
    Engine --> Runtime
    Engine --> Codec

    style Engine fill:#f96,stroke:#333,stroke-width:2px
    style Mailbox fill:#69f,stroke:#333,stroke-width:2px
    style Cache fill:#9f6,stroke:#333,stroke-width:2px
    style Metrics fill:#f69,stroke:#333,stroke-width:2px
```

### Container Descriptions

- **Engine**: Core processing unit that orchestrates all operations
- **Mailbox**: Public API interface for external communication
- **Message Cache**: Sophisticated storage system with LRU eviction
- **Metrics Module**: Telemetry and monitoring subsystem

---

## C4 Level 3: Component

The component diagram shows the internal structure of the Engine container and its key components.

```mermaid
graph TB
    subgraph "Engine Container"
        subgraph "Message Processing"
            Broadcast["Broadcast Handler<br/>- Send to recipients<br/>- Store locally"]
            Subscribe["Subscribe Handler<br/>- Waiter management<br/>- Instant delivery"]
            Get["Get Handler<br/>- Cache lookup<br/>- Batch retrieval"]
            Network["Network Handler<br/>- Receive messages<br/>- Decode & validate"]
        end

        subgraph "Cache Management"
            Deques["Peer Deques<br/>- LRU per peer<br/>- Size limited"]
            Items["Message Store<br/>- Commitment → Digest → Message<br/>- Deduplicated storage"]
            Counts["Reference Counter<br/>- Track multi-peer refs<br/>- Cleanup management"]
        end

        subgraph "Request Management"
            Waiters["Waiter Registry<br/>- Pending requests<br/>- Filter matching"]
            Cleanup["Cleanup Service<br/>- Dead waiter removal<br/>- Memory management"]
        end
    end

    Broadcast --> Items
    Broadcast --> Deques
    Subscribe --> Waiters
    Subscribe --> Items
    Get --> Items
    Get --> Deques
    Network --> Items
    Network --> Deques
    Network --> Counts
    Waiters --> Cleanup

    style Broadcast fill:#faa,stroke:#333,stroke-width:2px
    style Subscribe fill:#afa,stroke:#333,stroke-width:2px
    style Get fill:#aaf,stroke:#333,stroke-width:2px
    style Network fill:#ffa,stroke:#333,stroke-width:2px
```

### Component Interactions

- **Message Handlers**: Process different types of requests
- **Cache Components**: Manage message storage and eviction
- **Request Management**: Track pending requests and clean up resources

---

## C4 Level 4: Code

The code-level diagram shows the detailed class/struct relationships and key data structures.

```mermaid
classDiagram
    class Engine {
        -context: E
        -public_key: P
        -priority: bool
        -deque_size: usize
        -codec_config: M::Cfg
        -mailbox_receiver: Receiver
        -waiters: HashMap
        -items: HashMap
        -deques: HashMap
        -counts: HashMap
        -metrics: Metrics
        +new(context, config)
        +start(network)
        -run()
        -handle_broadcast()
        -handle_subscribe()
        -handle_get()
        -handle_network()
        -insert_message()
        -find_messages()
        -cleanup_waiters()
    }

    class Mailbox {
        -sender: mpsc::Sender
        +new(sender)
        +broadcast(recipients, message)
        +subscribe(peer, commitment, digest)
        +get(peer, commitment, digest)
    }

    class Message {
        <<enumeration>>
        Broadcast
        Subscribe
        Get
    }

    class Waiter {
        +peer: Option~P~
        +digest: Option~Dd~
        +responder: oneshot::Sender
    }

    class Pair {
        +commitment: Dc
        +digest: Dd
    }

    class Config {
        +public_key: P
        +mailbox_size: usize
        +deque_size: usize
        +priority: bool
        +codec_config: MCfg
    }

    class Metrics {
        +peer: Family
        +receive: Counter
        +subscribe: Counter
        +get: Counter
        +waiters: Gauge
    }

    Engine --> Mailbox : receives from
    Engine --> Message : processes
    Engine --> Waiter : manages
    Engine --> Pair : stores
    Engine --> Config : configured by
    Engine --> Metrics : reports to
    Mailbox --> Message : sends
```

### Key Data Structures

#### Message Storage Structure

```mermaid
graph LR
    subgraph "items: HashMap"
        C1["Commitment 1"] --> D1["Digest Map"]
        C2["Commitment 2"] --> D2["Digest Map"]
        D1 --> M1["Message 1"]
        D1 --> M2["Message 2"]
        D2 --> M3["Message 3"]
    end

    subgraph "deques: HashMap"
        P1["Peer 1"] --> DQ1["VecDeque<Pair>"]
        P2["Peer 2"] --> DQ2["VecDeque<Pair>"]
        DQ1 --> PA1["Pair(C1,D1)"]
        DQ1 --> PA2["Pair(C1,D2)"]
        DQ2 --> PA3["Pair(C2,D3)"]
    end

    subgraph "counts: HashMap"
        DG1["Digest 1"] --> CNT1["Count: 2"]
        DG2["Digest 2"] --> CNT2["Count: 1"]
    end
```

---

## Critical Process Flows

### 1. Message Broadcast Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant MB as Mailbox
    participant E as Engine
    participant C as Cache
    participant N as Network
    participant P as Peers

    App->>MB: broadcast(recipients, message)
    MB->>E: Message::Broadcast
    E->>C: insert_message(self_key, message)
    Note over C: Store in deques & items
    E->>N: send(recipients, message, priority)
    N->>P: Forward message
    E->>App: Return sent_to list

    Note over E: Handle potential cache eviction
    alt Cache full
        E->>C: Remove oldest from deque
        E->>C: Decrement reference count
        alt Count reaches 0
            E->>C: Remove from items
        end
    end
```

### 2. Message Subscribe Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant MB as Mailbox
    participant E as Engine
    participant C as Cache
    participant W as Waiters
    participant N as Network

    App->>MB: subscribe(peer, commitment, digest)
    MB->>E: Message::Subscribe

    alt Message in cache
        E->>C: find_messages()
        C-->>E: Return message
        E->>App: Send message immediately
    else Message not in cache
        E->>W: Add to waiters list
        Note over W: Store waiter with filters

        par Network receives message
            N->>E: Incoming message
            E->>C: insert_message()
            E->>W: Check waiters
            alt Waiter matches
                E->>App: Send message
                E->>W: Remove waiter
            end
        and Waiter timeout/cancel
            App->>W: Drop receiver
            E->>W: Cleanup dead waiters
        end
    end
```

### 3. Message Retrieval (Get) Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant MB as Mailbox
    participant E as Engine
    participant C as Cache

    App->>MB: get(peer, commitment, digest)
    MB->>E: Message::Get

    E->>C: find_messages(peer, commitment, digest, all=true)

    alt Peer filter provided
        Note over C: Search peer's deque only
        C->>C: Filter by commitment
        C->>C: Apply digest filter if provided
    else No peer filter
        Note over C: Search all messages
        C->>C: Get by commitment
        alt Digest provided
            C->>C: Return matching digest
        else No digest
            C->>C: Return all for commitment
        end
    end

    C-->>E: Return messages array
    E->>App: Send messages
```

### 4. Network Message Reception Flow

```mermaid
sequenceDiagram
    participant P as Remote Peer
    participant N as Network
    participant R as Receiver
    participant E as Engine
    participant C as Cache
    participant W as Waiters
    participant M as Metrics

    P->>N: Send message
    N->>R: Forward message
    R->>E: Decode message

    alt Decode success
        E->>C: insert_message(peer, message)

        alt Message new
            C->>C: Add to peer deque
            C->>C: Store in items
            C->>C: Increment ref count

            E->>W: Check waiters
            loop For each matching waiter
                E->>W: Send message
                E->>W: Remove fulfilled waiter
            end

            E->>M: receive.inc(Success)
        else Message duplicate
            C->>C: Move to front of deque
            E->>M: receive.inc(Dropped)
        end
    else Decode failure
        E->>M: receive.inc(Invalid)
    end
```

### 5. Cache Eviction Flow

```mermaid
flowchart TD
    Start([New Message Received])

    Start --> Check{Message in<br/>peer's deque?}

    Check -->|Yes| Move[Move to front<br/>of deque]
    Move --> End1([Return false<br/>duplicate])

    Check -->|No| Insert[Insert at front<br/>of deque]
    Insert --> IncCount[Increment<br/>reference count]

    IncCount --> FirstRef{Count == 1?}
    FirstRef -->|Yes| Store[Store message<br/>in items map]
    FirstRef -->|No| Skip1[Skip storage<br/>already exists]

    Store --> CheckSize{Deque size ><br/>max size?}
    Skip1 --> CheckSize

    CheckSize -->|No| End2([Return true<br/>inserted])
    CheckSize -->|Yes| Remove[Remove oldest<br/>from deque]

    Remove --> DecCount[Decrement<br/>reference count]
    DecCount --> LastRef{Count == 0?}

    LastRef -->|No| Skip2[Keep in items]
    LastRef -->|Yes| Delete[Remove from<br/>items map]

    Skip2 --> End3([Return true<br/>inserted])
    Delete --> End3
```

---

## Data Models

### Core Types

```mermaid
classDiagram
    class Committable {
        <<interface>>
        +commitment() Commitment
    }

    class Digestible {
        <<interface>>
        +digest() Digest
    }

    class Codec {
        <<interface>>
        +encode()
        +decode()
    }

    class PublicKey {
        <<interface>>
        +to_string() String
    }

    class Broadcaster {
        <<trait>>
        +Recipients
        +Message
        +Response
        +broadcast(recipients, message)
    }

    class TestMessage {
        +commitment: Vec~u8~
        +content: Vec~u8~
        +new(commitment, content)
        +shared(msg)
    }

    TestMessage ..|> Committable
    TestMessage ..|> Digestible
    TestMessage ..|> Codec
    Mailbox ..|> Broadcaster
```

### Message Type Hierarchy

```mermaid
graph TD
    Message[Message Enum]

    Message --> Broadcast["Broadcast<br/>- recipients: Recipients<br/>- message: M<br/>- responder: Sender"]
    Message --> Subscribe["Subscribe<br/>- peer: Option&lt;P&gt;<br/>- commitment: Commitment<br/>- digest: Option&lt;Digest&gt;<br/>- responder: Sender"]
    Message --> Get["Get<br/>- peer: Option&lt;P&gt;<br/>- commitment: Commitment<br/>- digest: Option&lt;Digest&gt;<br/>- responder: Sender"]
```

### Recipients Model

```mermaid
graph TD
    Recipients[Recipients Enum]

    Recipients --> All["All<br/>Broadcast to all peers"]
    Recipients --> One["One(PublicKey)<br/>Send to specific peer"]
    Recipients --> Many["Many(Vec&lt;PublicKey&gt;)<br/>Send to multiple peers"]
```

---

## Performance Characteristics

### Cache Complexity

| Operation          | Time Complexity           | Space Complexity  |
| ------------------ | ------------------------- | ----------------- |
| Insert Message     | O(1) amortized            | O(1)              |
| Find by Commitment | O(1)                      | -                 |
| Find by Peer       | O(n) where n = deque_size | -                 |
| Eviction           | O(1)                      | -                 |
| Reference Counting | O(1)                      | O(unique_digests) |

### Network Characteristics

```mermaid
graph LR
    subgraph "Message Flow Rates"
        B[Broadcast] -->|1:N| P[Peers]
        P -->|N:1| R[Receive]
        R -->|1:M| W[Waiters]
    end

    subgraph "Latency Factors"
        L1[Network Latency]
        L2[Codec Processing]
        L3[Cache Operations]
        L4[Waiter Matching]
    end
```

---

## Configuration Parameters

```mermaid
graph TD
    Config[Configuration]

    Config --> PK["public_key<br/>Node identity"]
    Config --> MS["mailbox_size<br/>Queue capacity"]
    Config --> DS["deque_size<br/>Cache per peer"]
    Config --> PR["priority<br/>Message priority"]
    Config --> CC["codec_config<br/>Serialization settings"]

    style Config fill:#f9f,stroke:#333,stroke-width:2px
```

### Tuning Guidelines

1. **mailbox_size**: Set based on expected message rate and processing capacity
2. **deque_size**: Balance between memory usage and cache hit rate
3. **priority**: Enable for time-sensitive broadcasts
4. **codec_config**: Configure based on message size and complexity

---

## Metrics and Observability

```mermaid
graph TB
    subgraph "Metric Categories"
        Peer["peer<br/>Messages per peer"]
        Receive["receive<br/>Success/Failure/Dropped"]
        Subscribe["subscribe<br/>Request outcomes"]
        Get["get<br/>Retrieval statistics"]
        Waiters["waiters<br/>Pending requests"]
    end

    subgraph "Status Labels"
        Success["Success"]
        Failure["Failure"]
        Dropped["Dropped"]
        Invalid["Invalid"]
    end

    Receive --> Success
    Receive --> Dropped
    Receive --> Invalid
    Subscribe --> Success
    Subscribe --> Dropped
    Get --> Success
    Get --> Failure
    Get --> Dropped
```

---

## Error Handling

```mermaid
flowchart TD
    E[Error Sources]

    E --> N[Network Errors]
    E --> C[Codec Errors]
    E --> M[Mailbox Errors]
    E --> T[Timeout Errors]

    N --> NR[Recovery: Log & Continue]
    C --> CR[Recovery: Drop Message]
    M --> MR[Recovery: Panic/Shutdown]
    T --> TR[Recovery: Clean Waiters]

    style E fill:#f99,stroke:#333,stroke-width:2px
```

---

## Testing Architecture

The codebase includes comprehensive tests demonstrating:

1. **Basic Operations**: Broadcast, subscribe, get
2. **Cache Management**: Eviction, multi-peer references
3. **Network Conditions**: Packet loss, selective recipients
4. **Edge Cases**: Self-retrieval, digest filtering
5. **Performance**: Cache efficiency, waiter cleanup

```mermaid
graph LR
    subgraph "Test Infrastructure"
        Sim[Simulated Network]
        Det[Deterministic Runtime]
        Mock[Mock Messages]
    end

    subgraph "Test Scenarios"
        T1[Broadcast Tests]
        T2[Cache Tests]
        T3[Network Tests]
        T4[Filter Tests]
    end

    Sim --> T1
    Sim --> T2
    Sim --> T3
    Det --> T4
    Mock --> T1
    Mock --> T2
```

---

## Summary

The `commonware-broadcast` system implements a sophisticated peer-to-peer message broadcasting system with:

- **Efficient Caching**: LRU per-peer with reference counting
- **Flexible Filtering**: By peer, commitment, and digest
- **Async Operations**: Non-blocking throughout
- **Robust Error Handling**: Graceful degradation
- **Comprehensive Metrics**: Full observability

The architecture follows clean separation of concerns with clear boundaries between network interaction, cache management, and request handling, making it suitable for distributed systems requiring reliable message dissemination.
