# Commonware Blockchain Architecture

## System Overview

The Commonware blockchain modules extend the existing Commonware distributed systems library to provide a complete blockchain platform. The architecture follows Commonware's core principles of composable primitives, runtime abstraction, and deterministic testing.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Application Layer                            │
├─────────────────────────────────────────────────────────────────┤
│  commonware-wallet  │  commonware-rpc   │  commonware-node     │
├─────────────────────────────────────────────────────────────────┤
│                    Blockchain Layer                             │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│commonware-   │commonware-   │commonware-   │  commonware-      │
│address       │transaction   │state         │  block            │
├──────────────┼──────────────┼──────────────┼───────────────────┤
│commonware-   │commonware-   │              │                   │
│economics     │sync          │              │                   │
├─────────────────────────────────────────────────────────────────┤
│             Commonware Foundation (includes BFT mempool)        │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│commonware-   │commonware-   │commonware-   │  commonware-      │
│consensus     │cryptography  │p2p           │  storage          │
│(ordered_     │              │              │                   │
│broadcast)    │              │              │                   │
├──────────────┼──────────────┼──────────────┼───────────────────┤
│commonware-   │commonware-   │commonware-   │  commonware-      │
│runtime       │broadcast     │codec         │  stream           │
└──────────────┴──────────────┴──────────────┴───────────────────┘
```

## Component Interactions

### Data Flow Architecture

```
Transaction Submission Flow (with ordered_broadcast):
User → Wallet → RPC → Node → ordered_broadcast (BFT ordering) → Block → State

Block Validation Flow:
Network → P2P → Sync → Block Validator → State Machine → Storage

State Query Flow:
RPC → State Machine → Storage → Merkle Proof → Response
```

### Module Dependencies

```
Foundation Layer (Commonware Core):
├── runtime (async execution)
├── cryptography (keys, signatures, hashes)
├── codec (serialization)
├── storage (persistence)
├── p2p (networking)
├── consensus (ordering)
└── broadcast (dissemination)

Blockchain Layer:
├── address (depends on: cryptography, codec)
├── transaction (depends on: address, cryptography, codec)
├── state (depends on: address, transaction, storage, codec)
├── block (depends on: transaction, state, cryptography, codec)
├── economics (depends on: transaction, state)
└── sync (depends on: block, state, p2p, consensus)

Transaction Pool Integration:
└── Uses ordered_broadcast from consensus (no separate mempool needed)

Application Layer:
├── wallet (depends on: address, transaction, cryptography)
├── rpc (depends on: all blockchain modules, p2p)
└── node (depends on: all modules)
```

## Core Design Patterns

### 1. Trait-Based Architecture

Every module defines clear trait boundaries:

```rust
// Each module provides abstract traits
pub trait Address: Send + Sync + Clone + PartialEq { }
pub trait Transaction: Send + Sync + Clone { }
pub trait StateMachine: Send + Sync { }

// Implementations are concrete types
pub struct Ed25519Address(pub [u8; 32]);
pub struct TransferTransaction { /* fields */ }
pub struct AccountStateMachine { /* fields */ }
```

**Benefits:**

- **Modularity**: Swap implementations without changing interfaces
- **Testing**: Mock implementations for isolated testing
- **Extensibility**: Add new transaction types or consensus algorithms
- **Composition**: Combine traits in different ways

### 2. Runtime Abstraction

All modules use Commonware's runtime abstraction:

```rust
// Never import tokio directly
// ❌ use tokio::time::sleep;

// Always use runtime capabilities
// ✅ context.sleep(duration).await;

pub trait StateMachine {
    type Context: storage::Context + Clone;

    async fn apply_transaction(
        &mut self,
        context: &Self::Context,
        tx: Transaction
    ) -> Result<Receipt, Error>;
}
```

**Benefits:**

- **Deterministic Testing**: All operations can be controlled in tests
- **Portability**: Run on different async runtimes
- **Controllability**: Precise control over timing and ordering

### 3. Deterministic Behavior

All components support deterministic execution:

```rust
#[test]
fn test_blockchain_behavior() {
    let executor = deterministic::Runner::seeded(42);
    executor.start(|context| async move {
        let mut blockchain = Blockchain::new(context.clone());

        // All operations are deterministic with same seed
        let result1 = blockchain.process_transaction(tx).await;

        // Test can be repeated with identical results
        let executor2 = deterministic::Runner::seeded(42);
        // ... will produce identical behavior
    });
}
```

**Benefits:**

- **Reproducible Testing**: Same inputs always produce same outputs
- **Debugging**: Replay exact scenarios for debugging
- **Auditing**: Verify system behavior matches specifications
- **Byzantine Testing**: Simulate complex network conditions

## State Management Architecture

### Account-Based Model

```
State Tree Structure:
├── Account States (address → Account)
│   ├── Balance
│   ├── Nonce
│   ├── Code Hash (for contracts)
│   └── Storage Root (for contract state)
├── Global State
│   ├── Block Height
│   ├── Total Supply
│   └── Network Parameters
└── Pending Changes
    ├── Transaction Queue
    └── State Deltas
```

### State Transitions

```rust
impl StateMachine {
    fn apply_transaction(&mut self, tx: Transaction) -> Result<Receipt, Error> {
        // 1. Validate transaction
        self.validate_transaction(&tx)?;

        // 2. Create state snapshot for rollback
        let snapshot = self.create_snapshot();

        // 3. Apply state changes
        match self.execute_transaction(&tx) {
            Ok(receipt) => {
                // Commit changes
                self.commit_changes();
                Ok(receipt)
            }
            Err(error) => {
                // Rollback to snapshot
                self.rollback_to_snapshot(snapshot);
                Err(error)
            }
        }
    }
}
```

## Consensus Integration

### Block Production Flow (with ordered_broadcast)

```
1. ordered_broadcast → BFT finalized transaction ordering
2. Block Builder → Construct block with ordered transactions  
3. State Machine → Execute transactions, generate receipts
4. Consensus → Block already has transaction ordering consensus
5. Validators → Validate state transitions and block structure
6. Finalization → Commit block and state changes
7. Network → Broadcast finalized block
```

### Integration with Commonware Consensus

```rust
impl ConsensusHandler for BlockchainNode {
    async fn on_proposal(&mut self, proposal: Proposal) -> Result<Vote, Error> {
        // 1. Decode block from proposal
        let block = Block::decode(&proposal.data)?;

        // 2. Validate block structure
        self.block_validator.validate(&block)?;

        // 3. Execute transactions to validate state transition
        let (new_state_root, receipts) = self.state_machine.apply_block(&block).await?;

        // 4. Verify state root matches block header
        if new_state_root != block.header.state_root {
            return Ok(Vote::Reject);
        }

        // 5. Vote to accept valid block
        Ok(Vote::Accept)
    }

    async fn on_finalization(&mut self, finalized: Finalized) -> Result<(), Error> {
        // Commit finalized block to storage
        let block = Block::decode(&finalized.data)?;
        self.storage.store_block(&block).await?;
        self.state_machine.finalize().await?;

        // ordered_broadcast automatically handles transaction cleanup
        // No manual mempool management needed

        Ok(())
    }
}
```

## Network Architecture

### P2P Communication Patterns

```
Transaction Gossip (via ordered_broadcast):
User → Node A → ordered_broadcast → BFT Consensus → [All Validators] → Ordered Transactions

Block Propagation:
Proposer → Consensus → Block → P2P → [Validator Nodes] → Validation

State Synchronization:
New Node → Request Headers → Request Blocks → Request State → Catch Up
```

### Message Types

```rust
#[derive(Encode, Decode)]
pub enum BlockchainMessage {
    // Transaction gossip
    NewTransaction(Transaction),
    TransactionRequest(Hash),

    // Block propagation
    NewBlock(Block),
    BlockRequest(Height),
    BlockResponse(Block),

    // State synchronization
    StateRequest { height: u64, root: Hash },
    StateResponse { proof: MerkleProof, data: Vec<u8> },

    // Transaction synchronization (handled by ordered_broadcast)
    TransactionSync { known_hashes: Vec<Hash> },
}
```

## Storage Architecture

### Data Layout

```
Storage Partitions:
├── blocks/           # Block data by height
│   ├── 0000000001   # Genesis block
│   ├── 0000000002   # Block 1
│   └── ...
├── state/           # Current state tree
│   ├── accounts/    # Account data
│   ├── merkle/      # Merkle tree nodes
│   └── metadata/    # Chain metadata
├── transactions/    # Transaction index
│   ├── by_hash/     # tx_hash → block_height
│   └── by_address/  # address → [tx_hashes]
└── ordered_broadcast/ # BFT transaction ordering (replaces traditional mempool)
    ├── chunks/      # Ordered transaction chunks
    ├── acks/        # Validator acknowledgments  
    └── locks/       # Threshold signatures
```

### Storage Operations

```rust
impl BlockchainStorage {
    // Block operations
    async fn store_block(&mut self, block: &Block) -> Result<(), Error>;
    async fn get_block(&self, height: u64) -> Result<Option<Block>, Error>;

    // State operations
    async fn get_account(&self, address: &Address) -> Result<Option<Account>, Error>;
    async fn set_account(&mut self, address: &Address, account: Account) -> Result<(), Error>;
    async fn get_state_root(&self) -> Result<Hash, Error>;

    // Transaction operations
    async fn store_transaction(&mut self, tx: &Transaction, block_height: u64) -> Result<(), Error>;
    async fn get_transaction(&self, hash: &Hash) -> Result<Option<Transaction>, Error>;

    // ordered_broadcast operations (BFT transaction ordering)
    async fn store_ordered_chunk(&mut self, chunk: &Chunk) -> Result<(), Error>;
    async fn get_finalized_transactions(&self, limit: usize) -> Result<Vec<Transaction>, Error>;
}
```

## Security Architecture

### Validation Layers

```
1. Format Validation → Syntax, encoding, size limits
2. Signature Validation → Cryptographic verification
3. State Validation → Balance, nonce, permissions
4. Economic Validation → Fees, gas limits, spam protection
5. Consensus Validation → Block ordering, finality
```

### Attack Resistance

```rust
impl SecurityLayer {
    // DoS protection
    fn rate_limit_transactions(&self, source: PeerId) -> bool;
    fn validate_gas_limits(&self, tx: &Transaction) -> bool;

    // Spam protection
    fn check_minimum_fee(&self, tx: &Transaction) -> bool;
    fn verify_account_nonce(&self, tx: &Transaction, account: &Account) -> bool;

    // Replay protection
    fn check_transaction_uniqueness(&self, tx: &Transaction) -> bool;
    fn validate_block_height(&self, block: &Block) -> bool;

    // Byzantine fault tolerance
    fn validate_consensus_messages(&self, msg: &ConsensusMessage) -> bool;
    fn handle_conflicting_blocks(&mut self, blocks: Vec<Block>) -> Resolution;
}
```

## Performance Characteristics

### Scalability Metrics

- **Transaction Throughput**: 1,000+ TPS target
- **Block Time**: 1-5 seconds configurable
- **Finality**: 1-3 blocks (Byzantine finality)
- **Storage Growth**: ~1GB per million transactions
- **Network Bandwidth**: ~100MB/day per node

### Optimization Strategies

1. **Parallel Processing**: Transaction validation in parallel
2. **Caching**: Frequently accessed state in memory
3. **Batch Operations**: Group database writes
4. **Compression**: Efficient block and state encoding
5. **Pruning**: Remove old state and transaction data

## Deployment Architecture

### Node Types

```
Full Node:
├── Complete blockchain history
├── Full state validation
├── Transaction mempool
├── Consensus participation
└── RPC API service

Light Node:
├── Block headers only
├── On-demand state queries
├── Transaction submission
└── Simplified validation

Archive Node:
├── Complete history preservation
├── Historical state queries
├── Block explorer services
└── Research and analytics
```

### Configuration Management

```rust
#[derive(Clone, Debug)]
pub struct NodeConfig {
    // Network configuration
    pub network: NetworkConfig,
    pub consensus: ConsensusConfig,

    // Storage configuration
    pub storage: StorageConfig,
    pub ordered_broadcast: OrderedBroadcastConfig,

    // API configuration
    pub rpc: RpcConfig,
    pub metrics: MetricsConfig,

    // Security configuration
    pub validation: ValidationConfig,
    pub rate_limiting: RateLimitConfig,
}
```

This architecture provides a solid foundation for building scalable, secure blockchain applications while leveraging Commonware's proven distributed systems primitives.
