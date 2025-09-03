# Commonware Blockchain Modules Specifications

## Module API Specifications

### 1. commonware-address

#### Core Types

```rust
/// Represents a blockchain address derived from cryptographic keys
pub trait Address:
    Send + Sync + Clone + PartialEq + Eq + Hash +
    Encode + Decode + Display + FromStr
{
    type PublicKey: PublicKey;
    type Error: Error + Send + Sync + 'static;

    /// Derive address from public key
    fn from_public_key(pk: &Self::PublicKey) -> Self;

    /// Get raw address bytes
    fn as_bytes(&self) -> &[u8];

    /// Create address from raw bytes
    fn from_bytes(bytes: &[u8]) -> Result<Self, Self::Error>;

    /// Validate address format and checksum
    fn validate(&self) -> bool;

    /// Get address length in bytes
    fn len(&self) -> usize;
}

/// Multi-signature address support
pub trait MultiSigAddress: Address {
    type Script: Encode + Decode;

    /// Create multi-sig address from script
    fn from_script(script: Self::Script) -> Self;

    /// Get underlying script
    fn script(&self) -> &Self::Script;

    /// Required signatures for spending
    fn required_signatures(&self) -> usize;
}
```

#### Implementation Example

```rust
/// Ed25519-based address implementation
#[derive(Clone, PartialEq, Eq, Hash, Encode, Decode)]
pub struct Ed25519Address([u8; 32]);

impl Address for Ed25519Address {
    type PublicKey = ed25519::PublicKey;
    type Error = AddressError;

    fn from_public_key(pk: &Self::PublicKey) -> Self {
        let hash = blake3::hash(pk.as_bytes());
        Self(hash.into())
    }

    fn as_bytes(&self) -> &[u8] {
        &self.0
    }

    fn from_bytes(bytes: &[u8]) -> Result<Self, Self::Error> {
        if bytes.len() != 32 {
            return Err(AddressError::InvalidLength);
        }
        let mut addr = [0u8; 32];
        addr.copy_from_slice(bytes);
        Ok(Self(addr))
    }

    fn validate(&self) -> bool {
        // Basic validation - all zeros is invalid
        self.0 != [0u8; 32]
    }

    fn len(&self) -> usize {
        32
    }
}
```

### 2. commonware-transaction

#### Core Types

```rust
/// Transaction trait for blockchain operations
pub trait Transaction:
    Send + Sync + Clone + PartialEq +
    Encode + Decode + Hash
{
    type Address: Address;
    type Signature: Signature;
    type Hash: Hash;
    type Error: Error + Send + Sync + 'static;

    /// Transaction sender
    fn from(&self) -> &Self::Address;

    /// Transaction recipient (None for contract creation)
    fn to(&self) -> Option<&Self::Address>;

    /// Transaction value/amount
    fn value(&self) -> u64;

    /// Transaction nonce (replay protection)
    fn nonce(&self) -> u64;

    /// Transaction fee
    fn fee(&self) -> u64;

    /// Transaction signature
    fn signature(&self) -> &Self::Signature;

    /// Transaction hash for identification
    fn hash(&self) -> Self::Hash;

    /// Verify transaction signature
    fn verify_signature(&self) -> bool;

    /// Get transaction size in bytes
    fn size(&self) -> usize;
}

/// Transaction validation trait
pub trait TransactionValidator {
    type Transaction: Transaction;
    type Context;
    type Error: Error + Send + Sync + 'static;

    /// Validate transaction against current state
    fn validate(
        &self,
        tx: &Self::Transaction,
        ctx: &Self::Context
    ) -> Result<(), Self::Error>;

    /// Check transaction format and signature
    fn validate_format(&self, tx: &Self::Transaction) -> Result<(), Self::Error>;

    /// Check transaction against state (balance, nonce)
    fn validate_state(
        &self,
        tx: &Self::Transaction,
        ctx: &Self::Context
    ) -> Result<(), Self::Error>;
}

/// Transaction execution trait
pub trait TransactionExecutor {
    type Transaction: Transaction;
    type State;
    type Receipt;
    type Error: Error + Send + Sync + 'static;

    /// Execute transaction and update state
    fn execute(
        &mut self,
        tx: &Self::Transaction,
        state: &mut Self::State,
    ) -> Result<Self::Receipt, Self::Error>;
}
```

#### Implementation Example

```rust
/// Basic transfer transaction
#[derive(Clone, PartialEq, Encode, Decode)]
pub struct TransferTransaction {
    pub from: Ed25519Address,
    pub to: Ed25519Address,
    pub value: u64,
    pub nonce: u64,
    pub gas_price: u64,
    pub gas_limit: u64,
    pub signature: ed25519::Signature,
    pub data: Vec<u8>, // Optional transaction data
}

impl Transaction for TransferTransaction {
    type Address = Ed25519Address;
    type Signature = ed25519::Signature;
    type Hash = blake3::Hash;
    type Error = TransactionError;

    fn from(&self) -> &Self::Address {
        &self.from
    }

    fn to(&self) -> Option<&Self::Address> {
        Some(&self.to)
    }

    fn value(&self) -> u64 {
        self.value
    }

    fn nonce(&self) -> u64 {
        self.nonce
    }

    fn fee(&self) -> u64 {
        self.gas_price * self.gas_limit
    }

    fn signature(&self) -> &Self::Signature {
        &self.signature
    }

    fn hash(&self) -> Self::Hash {
        let mut hasher = blake3::Hasher::new();
        hasher.update(&self.encode());
        hasher.finalize()
    }

    fn verify_signature(&self) -> bool {
        let message = self.signing_message();
        self.from.public_key().verify(&message, &self.signature)
    }

    fn size(&self) -> usize {
        self.encode().len()
    }
}
```

### 3. commonware-state

#### Core Types

```rust
/// Account state representation
#[derive(Clone, PartialEq, Encode, Decode)]
pub struct Account {
    pub balance: u64,
    pub nonce: u64,
    pub code_hash: Option<Hash>, // For smart contracts
    pub storage_root: Option<Hash>, // For contract storage
}

/// State machine trait for blockchain execution
pub trait StateMachine: Send + Sync {
    type Address: Address;
    type Transaction: Transaction;
    type Receipt;
    type Error: Error + Send + Sync + 'static;

    /// Get account state
    fn get_account(&self, address: &Self::Address) -> Option<Account>;

    /// Apply transaction to state
    fn apply_transaction(
        &mut self,
        tx: &Self::Transaction
    ) -> Result<Self::Receipt, Self::Error>;

    /// Get state root hash
    fn state_root(&self) -> Hash;

    /// Create state snapshot
    fn snapshot(&self) -> StateSnapshot;

    /// Revert to snapshot
    fn revert_to_snapshot(&mut self, snapshot: StateSnapshot);

    /// Finalize state changes
    fn finalize(&mut self);
}

/// State storage trait
pub trait StateStorage: Send + Sync {
    type Address: Address;
    type Error: Error + Send + Sync + 'static;

    /// Get account from storage
    fn get_account(&self, address: &Self::Address) -> Result<Option<Account>, Self::Error>;

    /// Set account in storage
    fn set_account(&mut self, address: &Self::Address, account: Account) -> Result<(), Self::Error>;

    /// Delete account from storage
    fn delete_account(&mut self, address: &Self::Address) -> Result<(), Self::Error>;

    /// Get storage root hash
    fn root_hash(&self) -> Result<Hash, Self::Error>;

    /// Create checkpoint for rollback
    fn checkpoint(&mut self) -> Result<CheckpointId, Self::Error>;

    /// Rollback to checkpoint
    fn rollback(&mut self, checkpoint: CheckpointId) -> Result<(), Self::Error>;

    /// Commit changes to storage
    fn commit(&mut self) -> Result<(), Self::Error>;
}
```

### 4. commonware-block

#### Core Types

```rust
/// Block header containing metadata
#[derive(Clone, PartialEq, Encode, Decode)]
pub struct BlockHeader {
    pub previous_hash: Hash,
    pub merkle_root: Hash,
    pub state_root: Hash,
    pub timestamp: u64,
    pub height: u64,
    pub gas_limit: u64,
    pub gas_used: u64,
    pub extra_data: Vec<u8>,
}

/// Complete block with header and transactions
#[derive(Clone, PartialEq, Encode, Decode)]
pub struct Block<T: Transaction> {
    pub header: BlockHeader,
    pub transactions: Vec<T>,
}

/// Block validation trait
pub trait BlockValidator {
    type Block;
    type Error: Error + Send + Sync + 'static;

    /// Validate block structure and contents
    fn validate_block(&self, block: &Self::Block) -> Result<(), Self::Error>;

    /// Validate block header
    fn validate_header(&self, header: &BlockHeader) -> Result<(), Self::Error>;

    /// Validate transactions in block
    fn validate_transactions(&self, block: &Self::Block) -> Result<(), Self::Error>;

    /// Check block links to parent correctly
    fn validate_parent_link(&self, block: &Self::Block, parent: &Self::Block) -> Result<(), Self::Error>;
}

/// Block builder for constructing new blocks
pub trait BlockBuilder {
    type Transaction: Transaction;
    type Block;
    type Error: Error + Send + Sync + 'static;

    /// Create new block builder
    fn new(parent_hash: Hash, height: u64) -> Self;

    /// Add transaction to block
    fn add_transaction(&mut self, tx: Self::Transaction) -> Result<(), Self::Error>;

    /// Set block timestamp
    fn set_timestamp(&mut self, timestamp: u64);

    /// Set extra data
    fn set_extra_data(&mut self, data: Vec<u8>);

    /// Build final block
    fn build(self, state_root: Hash) -> Result<Self::Block, Self::Error>;
}
```

### 5. Integration with ordered_broadcast (Replaces Traditional Mempool)

> **Architecture Note:** Instead of implementing a separate mempool module, we leverage Commonware's existing `ordered_broadcast` component which provides Byzantine fault-tolerant transaction ordering, gossip, and pool management in a single integrated solution.

#### Core Integration Traits

```rust
/// Blockchain implementation of Automaton trait for ordered_broadcast
pub trait BlockchainAutomaton: Automaton {
    type Address: Address;
    type Transaction: Transaction<Address = Self::Address>;
    type State: StateMachine;
    
    /// Context maps to (sequencer=address, height=nonce) 
    type Context = Context<Self::Address>;
    /// Digest maps to transaction hash
    type Digest = <Self::Transaction as Transaction>::Hash;
}

/// Implementation for blockchain transaction validation
impl Automaton for BlockchainNode {
    type Context = Context<Address>; // (account_address, nonce)
    type Digest = TransactionHash;

    async fn propose(&mut self, context: Self::Context) -> Receiver<TransactionHash> {
        // Get next pending transaction for this account at this nonce
        let (account, nonce) = (context.sequencer, context.height);
        self.get_pending_transaction_for_account(account, nonce).await
    }

    async fn verify(&mut self, context: Self::Context, tx_hash: TransactionHash) -> Receiver<bool> {
        // Validate transaction: signature, balance, nonce, gas limits
        let transaction = self.get_transaction(tx_hash).await?;
        self.validate_transaction_comprehensive(&transaction, &context).await
    }
}

/// Relay trait implementation for full transaction broadcasting
impl Relay for BlockchainRelay {
    type Digest = TransactionHash;
    
    async fn broadcast(&mut self, tx_hash: TransactionHash) {
        // Broadcast full transaction content over P2P network
        if let Some(transaction) = self.get_full_transaction(tx_hash).await {
            self.p2p_network.broadcast_transaction(transaction).await;
        }
    }
}

/// Configuration for blockchain-specific ordered_broadcast usage
pub struct BlockchainOrderedBroadcastConfig {
    /// Core ordered_broadcast config
    pub ordered_broadcast: ordered_broadcast::Config,
    
    /// Blockchain-specific settings
    pub max_transactions_per_account: u64,    // Prevent spam from single account
    pub fee_priority_threshold: u64,          // Minimum fee for priority handling
    pub transaction_ttl: Duration,            // Transaction time-to-live
    pub gas_price_oracle: Box<dyn GasPriceOracle>, // Dynamic fee estimation
}
```

#### Benefits of ordered_broadcast Integration

- **Byzantine Fault Tolerance**: Handles up to f malicious validators
- **Deterministic Ordering**: Eliminates MEV and front-running attacks  
- **Built-in Persistence**: Crash recovery without additional complexity
- **Threshold Cryptography**: Cryptographic proofs of transaction finality
- **Automatic Pruning**: Memory management with configurable bounds
- **Network Resilience**: Graceful handling of partitions and failures

## Integration Patterns

### Module Composition

```rust
/// Example of how modules work together with ordered_broadcast
pub struct BlockchainNode<A, T, S, B>
where
    A: Address,
    T: Transaction<Address = A>,
    S: StateMachine<Address = A, Transaction = T>,
    B: Block<T>,
{
    state: S,
    ordered_broadcast: Engine, // Replaces separate mempool + provides BFT ordering
    network: Box<dyn Network>,
    storage: Box<dyn Storage>,
}

impl<A, T, S, B> BlockchainNode<A, T, S, B>
where
    A: Address,
    T: Transaction<Address = A>,
    S: StateMachine<Address = A, Transaction = T>,
    B: Block<T>,
{
    /// Process incoming transaction via ordered_broadcast
    pub async fn handle_transaction(&mut self, tx: T) -> Result<(), Error> {
        // 1. Validate transaction format and signature
        if !tx.verify_signature() {
            return Err(Error::InvalidSignature);
        }

        // 2. Submit to ordered_broadcast for BFT ordering
        // The ordered_broadcast engine handles:
        // - Per-account nonce ordering
        // - Byzantine fault tolerance  
        // - Network gossip and persistence
        let context = Context {
            sequencer: tx.from().clone(), // Account address as sequencer
            height: tx.nonce(),           // Transaction nonce as height
        };
        
        self.ordered_broadcast.submit_transaction(context, tx.hash()).await?;
        
        Ok(())
    }

    /// Build block from ordered transactions
    pub async fn build_block_from_ordered_transactions(&mut self) -> Result<B, Error> {
        // 1. Get finalized transactions from ordered_broadcast
        // These are already ordered and have Byzantine agreement
        let ordered_tx_hashes = self.ordered_broadcast.get_finalized_transactions(1000).await;

        // 2. Build block with ordered transactions
        let mut builder = BlockBuilder::new(self.get_latest_hash(), self.get_height() + 1);
        for tx_hash in ordered_tx_hashes {
            let tx = self.get_full_transaction(tx_hash)?;
            builder.add_transaction(tx)?;
        }

        // 3. Execute transactions and get state root
        let state_root = self.execute_transactions(&builder.transactions())?;

        // 4. Build final block
        builder.build(state_root)
    }
}
```

### Testing Integration

```rust
#[cfg(test)]
mod tests {
    use commonware_runtime::deterministic;

    #[test]
    fn test_end_to_end_transaction_flow_with_ordered_broadcast() {
        let executor = deterministic::Runner::seeded(42);
        executor.start(|context| async move {
            // Setup blockchain components with ordered_broadcast
            let mut state = InMemoryStateMachine::new();
            let config = BlockchainOrderedBroadcastConfig::new(test_private_key());
            let mut ordered_broadcast = Engine::new(context.clone(), config);
            
            // Create test transaction
            let tx = create_test_transaction();
            let tx_context = Context {
                sequencer: tx.from().clone(),
                height: tx.nonce(),
            };
            
            // Test ordered_broadcast transaction flow
            assert!(tx.verify_signature());
            
            // Submit to ordered_broadcast (handles ordering + BFT)
            ordered_broadcast.submit_transaction(tx_context, tx.hash()).await.unwrap();
            
            // Wait for BFT finalization
            let finalized_hashes = ordered_broadcast.get_finalized_transactions(1).await;
            assert_eq!(finalized_hashes.len(), 1);
            assert_eq!(finalized_hashes[0], tx.hash());
            
            // Apply finalized transaction to state
            let receipt = state.apply_transaction(&tx).await.unwrap();
            assert_eq!(receipt.status, ExecutionStatus::Success);
            
            // Verify state changes
            let account = state.get_account(&tx.to().unwrap()).unwrap();
            assert_eq!(account.balance, 1000);
        });
    }
}
```

## Development Guidelines

### 1. Error Handling

- Use `thiserror` for all error types
- Provide detailed error context
- Implement `From` conversions for error chaining

### 2. Async Design

- All I/O operations must be async
- Use `futures` crate, not tokio directly
- Leverage commonware-runtime capabilities

### 3. Testing Strategy

- Use deterministic runtime for all async tests
- Aim for 90%+ test coverage
- Include property-based tests for critical algorithms
- Test Byzantine behavior and edge cases

### 4. Performance Considerations

- Minimize allocations in hot paths
- Use `Bytes` for zero-copy operations
- Implement efficient serialization
- Profile and benchmark critical operations

### 5. Security Requirements

- Validate all inputs rigorously
- Use constant-time operations for cryptographic code
- Implement proper nonce handling
- Test against known attack vectors

This specification provides the foundation for implementing a complete blockchain system using Commonware's proven distributed systems primitives.
