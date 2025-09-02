# P2P Network Architecture Documentation

## Table of Contents

1. [C4 Model Overview](#c4-model-overview)
2. [Level 1: System Context](#level-1-system-context)
3. [Level 2: Container Diagram](#level-2-container-diagram)
4. [Level 3: Component Diagrams](#level-3-component-diagrams)
5. [Level 4: Code-Level Details](#level-4-code-level-details)
6. [Critical Process Flows](#critical-process-flows)
7. [Architecture Decision Records](#architecture-decision-records)

---

## C4 Model Overview

This documentation describes the architecture of the commonware-p2p library using the C4 Model, providing four levels of abstraction from high-level system context down to code-level implementation details.

### Architecture Principles

- **Authenticated Communication**: All peer connections are cryptographically authenticated
- **Encrypted Transport**: All messages are encrypted in transit
- **Rate Limiting**: Per-channel and per-peer rate limiting to prevent abuse
- **Modular Design**: Clear separation between discovery and lookup mechanisms
- **Actor-Based Concurrency**: Uses actor pattern for managing concurrent operations

---

## Level 1: System Context

### Overview

The P2P system enables authenticated, encrypted communication between distributed peers in a network.

```mermaid
C4Context
    title System Context - P2P Network

    Person(peer, "Network Peer", "A node participating in the P2P network")
    System(p2p, "P2P Network Library", "Enables authenticated, encrypted peer-to-peer communication")
    System_Ext(blockchain, "Blockchain/Consensus", "External system providing peer authorization")
    System_Ext(application, "Application Layer", "Higher-level application using P2P communication")

    Rel(peer, p2p, "Connects and communicates", "Encrypted/Authenticated")
    Rel(p2p, blockchain, "Gets peer sets", "Authorization data")
    Rel(application, p2p, "Uses", "Message passing API")
```

### Key Actors

- **Network Peers**: Nodes that participate in the P2P network
- **Application Layer**: Higher-level applications that use the P2P library
- **External Authority**: Systems that provide peer authorization (e.g., blockchain staking sets)

---

## Level 2: Container Diagram

### Overview

The P2P system consists of multiple containers that handle different aspects of peer communication.

```mermaid
C4Container
    title Container Diagram - P2P Network Library

    Container(discovery, "Discovery Module", "Rust", "Automatic peer discovery using bit vectors")
    Container(lookup, "Lookup Module", "Rust", "Direct peer connections with known addresses")
    Container(stream, "Stream Layer", "commonware-stream", "Encrypted communication channels")
    Container(codec, "Codec Layer", "commonware-codec", "Message encoding/decoding")
    Container(runtime, "Runtime", "commonware-runtime", "Async execution and scheduling")

    Container_Ext(network, "Network Layer", "TCP/UDP", "Network transport")

    Rel(discovery, stream, "Uses", "Encrypted connections")
    Rel(lookup, stream, "Uses", "Encrypted connections")
    Rel(stream, codec, "Uses", "Message serialization")
    Rel(stream, network, "Uses", "Network I/O")
    Rel(discovery, runtime, "Runs on")
    Rel(lookup, runtime, "Runs on")
```

### Container Descriptions

| Container | Purpose                  | Technology         | Responsibilities                                         |
| --------- | ------------------------ | ------------------ | -------------------------------------------------------- |
| Discovery | Automatic peer discovery | Rust + Actors      | Peer discovery, bit vector gossip, connection management |
| Lookup    | Direct peer connections  | Rust + Actors      | Known peer connections, simplified routing               |
| Stream    | Encrypted channels       | commonware-stream  | Encryption, authentication, stream multiplexing          |
| Codec     | Message encoding         | commonware-codec   | Serialization, deserialization, wire format              |
| Runtime   | Async execution          | commonware-runtime | Task scheduling, I/O, timers                             |

---

## Level 3: Component Diagrams

### 3.1 Discovery Module Components

```mermaid
graph TB
    subgraph "Discovery Module"
        Tracker[Tracker Actor<br/>- Peer authorization<br/>- Connection reservations]
        Router[Router Actor<br/>- Message routing<br/>- Channel management]
        Dialer[Dialer Actor<br/>- Outbound connections<br/>- Connection attempts]
        Listener[Listener Actor<br/>- Inbound connections<br/>- Handshake handling]
        Spawner[Spawner Actor<br/>- Peer lifecycle<br/>- Actor management]
        Peer[Peer Actors<br/>- Per-peer communication<br/>- Rate limiting]

        Oracle[Oracle<br/>- Peer registration<br/>- Authorization updates]
        Channels[Channels<br/>- Application channels<br/>- Message queuing]
    end

    Oracle -->|Register peers| Tracker
    Tracker -->|Authorize| Listener
    Tracker -->|Provide peers| Dialer
    Listener -->|New connection| Spawner
    Dialer -->|New connection| Spawner
    Spawner -->|Create| Peer
    Peer -->|Messages| Router
    Router -->|Deliver| Channels
    Channels -->|To application| App[Application]
```

### 3.2 Lookup Module Components

```mermaid
graph TB
    subgraph "Lookup Module"
        LTracker[Tracker Actor<br/>- Simpler peer tracking<br/>- Direct addressing]
        LRouter[Router Actor<br/>- Message routing]
        LDialer[Dialer Actor<br/>- Direct dial]
        LListener[Listener Actor<br/>- Accept connections]
        LSpawner[Spawner Actor<br/>- Peer management]
        LPeer[Peer Actors<br/>- Ping/Data messages]

        LOracle[Oracle<br/>- Peer + Address registration]
        LChannels[Channels<br/>- Application channels]
    end

    LOracle -->|Register peers + addresses| LTracker
    LTracker -->|Authorize| LListener
    LTracker -->|Provide addresses| LDialer
    LListener -->|New connection| LSpawner
    LDialer -->|New connection| LSpawner
    LSpawner -->|Create| LPeer
    LPeer -->|Messages| LRouter
    LRouter -->|Deliver| LChannels
```

### 3.3 Message Types and Flow

```mermaid
classDiagram
    class Message {
        <<enumeration>>
        +Ping
        +Data(channel, bytes)
        +BitVec(index, bits)
        +Peers(Vec~PeerInfo~)
    }

    class PeerInfo {
        +socket: SocketAddr
        +timestamp: u64
        +public_key: PublicKey
        +signature: Signature
        +verify() bool
    }

    class BitVec {
        +index: u64
        +bits: BitVector
    }

    class Data {
        +channel: u32
        +message: Bytes
    }

    Message --> Data : contains
    Message --> BitVec : contains
    Message --> PeerInfo : contains
```

---

## Level 4: Code-Level Details

### 4.1 Actor Architecture

```mermaid
sequenceDiagram
    participant App as Application
    participant Oracle as Oracle
    participant Tracker as Tracker Actor
    participant Dialer as Dialer Actor
    participant Spawner as Spawner Actor
    participant Peer as Peer Actor
    participant Router as Router Actor

    App->>Oracle: register(peers)
    Oracle->>Tracker: Register message

    loop Dial attempts
        Dialer->>Tracker: dialable()
        Tracker-->>Dialer: peer list
        Dialer->>Tracker: dial(peer)
        Tracker-->>Dialer: Reservation
        Dialer->>Spawner: spawn(connection)
        Spawner->>Peer: create
        Peer->>Tracker: connect(peer)
        Peer->>Router: ready(relay)
    end
```

### 4.2 Connection State Machine

```mermaid
stateDiagram-v2
    [*] --> Inert: Initial
    Inert --> Reserved: reserve()
    Reserved --> Active: connect()
    Active --> Inert: release()
    Reserved --> Inert: release()

    state Reserved {
        [*] --> Dialing: As Dialer
        [*] --> Listening: As Listener
    }

    state Active {
        [*] --> Connected
        Connected --> Sending: send()
        Sending --> Connected
        Connected --> Receiving: recv()
        Receiving --> Connected
    }
```

### 4.3 Rate Limiting Implementation

```mermaid
graph LR
    subgraph "Rate Limiter"
        Check[Check Rate]
        Allow[Allow Request]
        Deny[Deny Request]
        Wait[Wait for Reset]
    end

    Request -->|Incoming| Check
    Check -->|Under limit| Allow
    Check -->|Over limit| Deny
    Deny --> Wait
    Wait -->|Time elapsed| Check
    Allow -->|Process| Response
```

---

## Critical Process Flows

### 5.1 Peer Discovery Flow (Discovery Module)

```mermaid
sequenceDiagram
    participant P1 as Peer 1
    participant P2 as Peer 2
    participant P3 as Peer 3

    Note over P1,P3: Initial Bootstrap
    P1->>P2: Connect (bootstrapper)
    P1->>P2: Send PeerInfo (signed)
    P1->>P2: Send BitVec [1,0,0]

    Note over P2: P2 knows P3
    P2->>P1: Send Peers [P3 info]

    Note over P1: Learn about P3
    P1->>P3: Dial using P3 info
    P3->>P1: Accept connection

    Note over P1,P3: Gossip continues
    P1->>P2: Send BitVec [1,1,1]
    P2->>P3: Send BitVec [1,1,1]
```

### 5.2 Message Routing Flow

```mermaid
flowchart TB
    Start([Message from App]) --> Sender[Sender.send]
    Sender --> CheckSize{Size OK?}
    CheckSize -->|No| Error1[MessageTooLarge]
    CheckSize -->|Yes| Router[Router Actor]

    Router --> Recipients{Recipients?}
    Recipients -->|One| SendOne[Send to specific peer]
    Recipients -->|Some| SendSome[Send to peer list]
    Recipients -->|All| SendAll[Broadcast to all]

    SendOne --> Relay[Peer Relay]
    SendSome --> Relay
    SendAll --> Relay

    Relay --> Priority{Priority?}
    Priority -->|High| HighQueue[High Priority Queue]
    Priority -->|Low| LowQueue[Low Priority Queue]

    HighQueue --> PeerActor[Peer Actor]
    LowQueue --> PeerActor

    PeerActor --> RateLimit{Rate OK?}
    RateLimit -->|No| Drop[Drop Message]
    RateLimit -->|Yes| Encode[Encode Message]

    Encode --> Stream[Stream Layer]
    Stream --> Network[Network Send]
```

### 5.3 Connection Establishment Flow

```mermaid
sequenceDiagram
    participant Dialer
    participant Network
    participant Listener
    participant Tracker
    participant Spawner

    Note over Dialer: Initiate connection
    Dialer->>Tracker: Request reservation
    Tracker->>Tracker: Check authorization
    Tracker-->>Dialer: Reservation granted

    Dialer->>Network: TCP connect
    Network->>Listener: Accept connection

    Note over Dialer,Listener: Handshake
    Dialer->>Listener: Send public key
    Listener->>Tracker: listenable(peer)?
    Tracker-->>Listener: true

    Listener->>Dialer: Send public key
    Dialer->>Listener: Complete handshake

    Note over Listener: Spawn peer
    Listener->>Tracker: listen(peer)
    Tracker-->>Listener: Reservation
    Listener->>Spawner: spawn(connection)

    Note over Dialer: Spawn peer
    Dialer->>Spawner: spawn(connection)

    Spawner->>Spawner: Create Peer Actor
    Spawner->>Tracker: connect(peer)
```

### 5.4 Peer Authorization Update Flow

```mermaid
flowchart LR
    subgraph "External Authority"
        Blockchain[Blockchain/Consensus]
    end

    subgraph "Application"
        App[Application Layer]
    end

    subgraph "P2P Library"
        Oracle[Oracle]
        Tracker[Tracker]
        Peers[Active Peers]
    end

    Blockchain -->|New peer set| App
    App -->|register(index, peers)| Oracle
    Oracle -->|Update| Tracker
    Tracker -->|Check active| Peers
    Tracker -->|Kill unauthorized| Peers
    Tracker -->|Update routing| Tracker
```

---

## Architecture Decision Records

### ADR-001: Actor-Based Architecture

**Status**: Accepted

**Context**: Need to manage concurrent operations for multiple peers with independent state.

**Decision**: Use actor pattern with message passing for concurrency control.

**Consequences**:

- ✅ Simplified reasoning about concurrent state
- ✅ Natural isolation between peer connections
- ✅ Easy to add new actor types
- ❌ Message passing overhead
- ❌ Debugging can be more complex

### ADR-002: Separate Discovery and Lookup Modules

**Status**: Accepted

**Context**: Different use cases require different peer discovery mechanisms.

**Decision**: Implement two separate modules:

- Discovery: For networks where peer addresses are unknown
- Lookup: For networks where peer addresses are known

**Consequences**:

- ✅ Optimized for specific use cases
- ✅ Simpler implementation for each module
- ✅ Can choose appropriate module
- ❌ Some code duplication
- ❌ Two APIs to maintain

### ADR-003: Rate Limiting Strategy

**Status**: Accepted

**Context**: Need to prevent DoS attacks and resource exhaustion.

**Decision**: Implement multi-level rate limiting:

- Per-peer connection rate limiting
- Per-channel message rate limiting
- Global incoming connection rate limiting

**Consequences**:

- ✅ Fine-grained control over resource usage
- ✅ Protection against various attack vectors
- ✅ Fair resource allocation
- ❌ Additional complexity
- ❌ Potential for legitimate traffic to be limited

### ADR-004: Bit Vector Discovery Protocol

**Status**: Accepted

**Context**: Need efficient way to communicate peer knowledge in discovery module.

**Decision**: Use bit vectors indexed by peer set to communicate knowledge.

**Consequences**:

- ✅ Very bandwidth efficient
- ✅ O(n) bits for n peers
- ✅ Natural gossip protocol
- ❌ Requires synchronized peer set knowledge
- ❌ Limited to tracked peer sets

---

## Performance Characteristics

### Scalability Metrics

| Metric              | Discovery Module         | Lookup Module      |
| ------------------- | ------------------------ | ------------------ |
| Connection Overhead | O(n) for bit vectors     | O(1) direct dial   |
| Discovery Time      | O(log n) hops            | Immediate          |
| Memory per Peer     | ~1KB + channels          | ~500B + channels   |
| Message Overhead    | 11 bytes + payload       | 11 bytes + payload |
| Max Peer Sets       | Configurable (default 4) | Configurable       |
| Max Peers per Set   | 2^16 (65,536)            | Unlimited          |

### Resource Limits

```mermaid
graph TB
    subgraph "Resource Limits"
        ConnRate[Connection Rate<br/>1/min per peer]
        MsgRate[Message Rate<br/>Configurable per channel]
        MsgSize[Message Size<br/>Configurable max]
        Mailbox[Mailbox Size<br/>Default 1000]
        PeerSets[Tracked Sets<br/>Default 4]
    end

    subgraph "Protection Against"
        DoS[DoS Attacks]
        Flood[Message Flooding]
        Memory[Memory Exhaustion]
        CPU[CPU Exhaustion]
    end

    ConnRate --> DoS
    MsgRate --> Flood
    MsgSize --> Memory
    Mailbox --> Memory
    PeerSets --> Memory
```

---

## Security Model

### Trust Boundaries

```mermaid
graph TB
    subgraph "Trusted"
        App[Application]
        Oracle[Oracle]
        LocalActors[Local Actors]
    end

    subgraph "Verified"
        AuthPeers[Authorized Peers]
        SignedMsgs[Signed Messages]
    end

    subgraph "Untrusted"
        Network[Network Layer]
        UnknownPeers[Unknown Peers]
        RawMessages[Raw Messages]
    end

    UnknownPeers -->|Handshake| AuthPeers
    RawMessages -->|Verify| SignedMsgs
    Network -->|Encrypted| LocalActors
    AuthPeers -->|Rate Limited| LocalActors
    Oracle -->|Authorize| AuthPeers
```

### Security Properties

1. **Authentication**: All peers are cryptographically authenticated
2. **Encryption**: All messages are encrypted in transit
3. **Authorization**: Only authorized peers can connect
4. **Rate Limiting**: Protection against resource exhaustion
5. **Message Integrity**: All messages are authenticated
6. **Replay Protection**: Timestamps and nonces prevent replay attacks

---

## Deployment Considerations

### Configuration Parameters

| Parameter                          | Description        | Recommended Production | Aggressive (Demo)    |
| ---------------------------------- | ------------------ | ---------------------- | -------------------- |
| `synchrony_bound`                  | Max clock skew     | 5 seconds              | 5 seconds            |
| `handshake_timeout`                | Connection timeout | 5 seconds              | 5 seconds            |
| `allowed_connection_rate_per_peer` | Per-peer rate      | 1/minute               | 1/second             |
| `dial_frequency`                   | Dial attempt rate  | 1 second               | 500ms                |
| `query_frequency`                  | Peer list refresh  | 60 seconds             | 30 seconds           |
| `gossip_bit_vec_frequency`         | Discovery gossip   | 50 seconds             | 5 seconds            |
| `max_message_size`                 | Maximum message    | Application-specific   | Application-specific |

### Network Requirements

- **Bandwidth**: Minimal overhead (~11 bytes per message + encryption)
- **Latency**: Configurable timeouts, typically 5-60 seconds
- **Connectivity**: Supports NAT traversal through peer introduction
- **Ports**: Single listening port per node

---

## Conclusion

The commonware-p2p library provides a robust, scalable, and secure foundation for peer-to-peer networking with:

- **Modular architecture** supporting different discovery mechanisms
- **Actor-based concurrency** for scalable peer management
- **Multi-layered security** with authentication, encryption, and rate limiting
- **Efficient protocols** for peer discovery and message routing
- **Flexible configuration** for different deployment scenarios

The architecture supports both small private networks and large public networks with thousands of peers, while maintaining security and performance characteristics suitable for production blockchain and distributed systems.
