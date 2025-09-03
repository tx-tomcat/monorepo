# Getting Started with Commonware Blockchain Modules

## Quick Start Guide

This guide helps you get started with implementing blockchain functionality using Commonware's modular architecture.

## Prerequisites

1. **Rust Environment**

   ```bash
   # Install Rust (if not already installed)
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

   # Install nightly toolchain (for formatting)
   rustup toolchain install nightly
   rustup component add rustfmt --toolchain nightly
   ```

2. **Commonware Repository**

   ```bash
   # Clone the repository
   git clone https://github.com/commonwarexyz/monorepo.git
   cd monorepo

   # Verify build works
   cargo build --workspace --all-targets
   ```

## Phase 1: Implementing commonware-address

### Step 1: Create the Module Structure

```bash
# Create the address module directory
mkdir -p commonware-address/src
mkdir -p commonware-address/examples
mkdir -p commonware-address/tests

# Create Cargo.toml
cat > commonware-address/Cargo.toml << 'EOF'
[package]
name = "commonware-address"
version = "0.0.1"
edition = "2021"

[dependencies]
commonware-cryptography = { path = "../cryptography" }
commonware-codec = { path = "../codec" }
blake3 = "1.5"
thiserror = "1.0"
serde = { version = "1.0", features = ["derive"] }

[dev-dependencies]
commonware-runtime = { path = "../runtime" }
hex = "0.4"
EOF
```

### Step 2: Implement Basic Address Types

Create `commonware-address/src/lib.rs`:

````rust
//! # commonware-address
//!
//! Address management and derivation for blockchain applications.
//!
//! ## Status
//!
//! This crate is under active development as part of the Commonware blockchain modules.
//!
//! ## Examples
//!
//! ```rust
//! use commonware_address::{Address, Ed25519Address};
//! use commonware_cryptography::ed25519;
//!
//! // Generate a key pair
//! let (private_key, public_key) = ed25519::generate();
//!
//! // Derive address from public key
//! let address = Ed25519Address::from_public_key(&public_key);
//!
//! // Convert to/from bytes
//! let bytes = address.as_bytes();
//! let recovered = Ed25519Address::from_bytes(bytes).unwrap();
//! assert_eq!(address, recovered);
//! ```

use std::fmt::{Display, Formatter, Result as FmtResult};
use std::str::FromStr;
use std::hash::{Hash, Hasher};

use commonware_cryptography::{PublicKey, ed25519};
use commonware_codec::{Encode, Decode};
use thiserror::Error;

pub mod ed25519_address;
pub mod multisig;

pub use ed25519_address::Ed25519Address;

/// Address trait for blockchain addresses
pub trait Address:
    Send + Sync + Clone + PartialEq + Eq + Hash +
    Encode + Decode + Display + FromStr
{
    type PublicKey: PublicKey;
    type Error: std::error::Error + Send + Sync + 'static;

    /// Derive address from public key
    fn from_public_key(pk: &Self::PublicKey) -> Self;

    /// Get raw address bytes
    fn as_bytes(&self) -> &[u8];

    /// Create address from raw bytes
    fn from_bytes(bytes: &[u8]) -> Result<Self, Self::Error>;

    /// Validate address format
    fn validate(&self) -> bool;

    /// Get address length in bytes
    fn len(&self) -> usize;

    /// Check if address is empty/zero
    fn is_empty(&self) -> bool {
        self.as_bytes().iter().all(|&b| b == 0)
    }
}

/// Common address errors
#[derive(Error, Debug, PartialEq)]
pub enum AddressError {
    #[error("invalid address length: expected {expected}, got {actual}")]
    InvalidLength { expected: usize, actual: usize },

    #[error("invalid address format: {0}")]
    InvalidFormat(String),

    #[error("invalid checksum")]
    InvalidChecksum,

    #[error("empty address not allowed")]
    EmptyAddress,
}
````

### Step 3: Implement Ed25519 Address

Create `commonware-address/src/ed25519_address.rs`:

```rust
use std::fmt::{Display, Formatter, Result as FmtResult};
use std::str::FromStr;
use std::hash::{Hash, Hasher};

use commonware_cryptography::ed25519;
use commonware_codec::{Encode, Decode};

use crate::{Address, AddressError};

/// Ed25519-based blockchain address
#[derive(Clone, PartialEq, Eq)]
pub struct Ed25519Address([u8; 32]);

impl Ed25519Address {
    /// Address length in bytes
    pub const LENGTH: usize = 32;

    /// Create zero address (for testing)
    pub fn zero() -> Self {
        Self([0u8; 32])
    }

    /// Create address from raw bytes (unchecked)
    pub fn from_bytes_unchecked(bytes: [u8; 32]) -> Self {
        Self(bytes)
    }
}

impl Address for Ed25519Address {
    type PublicKey = ed25519::PublicKey;
    type Error = AddressError;

    fn from_public_key(pk: &Self::PublicKey) -> Self {
        let hash = blake3::hash(pk.as_ref());
        Self(hash.into())
    }

    fn as_bytes(&self) -> &[u8] {
        &self.0
    }

    fn from_bytes(bytes: &[u8]) -> Result<Self, Self::Error> {
        if bytes.len() != 32 {
            return Err(AddressError::InvalidLength {
                expected: 32,
                actual: bytes.len()
            });
        }

        let mut addr = [0u8; 32];
        addr.copy_from_slice(bytes);

        let address = Self(addr);
        if !address.validate() {
            return Err(AddressError::EmptyAddress);
        }

        Ok(address)
    }

    fn validate(&self) -> bool {
        // Basic validation - all zeros is invalid
        !self.is_empty()
    }

    fn len(&self) -> usize {
        32
    }
}

impl Display for Ed25519Address {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        write!(f, "0x{}", hex::encode(&self.0))
    }
}

impl FromStr for Ed25519Address {
    type Err = AddressError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let s = s.strip_prefix("0x").unwrap_or(s);
        let bytes = hex::decode(s).map_err(|e| {
            AddressError::InvalidFormat(format!("hex decode error: {}", e))
        })?;

        Self::from_bytes(&bytes)
    }
}

impl Hash for Ed25519Address {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.0.hash(state);
    }
}

impl Encode for Ed25519Address {
    fn encode(&self) -> Vec<u8> {
        self.0.to_vec()
    }
}

impl Decode for Ed25519Address {
    type Error = AddressError;

    fn decode(bytes: &[u8]) -> Result<Self, Self::Error> {
        Self::from_bytes(bytes)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use commonware_runtime::deterministic;

    #[test]
    fn test_address_from_public_key() {
        let executor = deterministic::Runner::default();
        executor.start(|_context| async move {
            // Generate key pair
            let (_, public_key) = ed25519::generate();

            // Derive address
            let address = Ed25519Address::from_public_key(&public_key);

            // Verify properties
            assert!(address.validate());
            assert_eq!(address.len(), 32);
            assert!(!address.is_empty());
        });
    }

    #[test]
    fn test_address_serialization() {
        let executor = deterministic::Runner::default();
        executor.start(|_context| async move {
            let (_, public_key) = ed25519::generate();
            let address = Ed25519Address::from_public_key(&public_key);

            // Test bytes round trip
            let bytes = address.as_bytes();
            let recovered = Ed25519Address::from_bytes(bytes).unwrap();
            assert_eq!(address, recovered);

            // Test string round trip
            let string = address.to_string();
            let parsed = Ed25519Address::from_str(&string).unwrap();
            assert_eq!(address, parsed);

            // Test encode/decode
            let encoded = address.encode();
            let decoded = Ed25519Address::decode(&encoded).unwrap();
            assert_eq!(address, decoded);
        });
    }

    #[test]
    fn test_invalid_addresses() {
        // Test invalid length
        let result = Ed25519Address::from_bytes(&[0u8; 31]);
        assert!(matches!(result, Err(AddressError::InvalidLength { .. })));

        // Test empty address
        let result = Ed25519Address::from_bytes(&[0u8; 32]);
        assert!(matches!(result, Err(AddressError::EmptyAddress)));

        // Test invalid hex string
        let result = Ed25519Address::from_str("invalid_hex");
        assert!(matches!(result, Err(AddressError::InvalidFormat(_))));
    }
}
```

### Step 4: Create Example

Create `commonware-address/examples/basic_usage.rs`:

```rust
//! Basic usage example for commonware-address
//!
//! Run with: cargo run --example basic_usage

use commonware_address::{Address, Ed25519Address};
use commonware_cryptography::ed25519;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== Commonware Address Example ===\n");

    // Generate a key pair
    println!("1. Generating Ed25519 key pair...");
    let (private_key, public_key) = ed25519::generate();
    println!("   Public key: {}", hex::encode(public_key.as_ref()));

    // Derive address from public key
    println!("\n2. Deriving address from public key...");
    let address = Ed25519Address::from_public_key(&public_key);
    println!("   Address: {}", address);
    println!("   Address bytes: {}", hex::encode(address.as_bytes()));

    // Validate address
    println!("\n3. Validating address...");
    println!("   Valid: {}", address.validate());
    println!("   Length: {} bytes", address.len());
    println!("   Is empty: {}", address.is_empty());

    // Test serialization
    println!("\n4. Testing serialization...");

    // Bytes round trip
    let bytes = address.as_bytes();
    let recovered_from_bytes = Ed25519Address::from_bytes(bytes)?;
    println!("   Bytes round trip: {}", address == recovered_from_bytes);

    // String round trip
    let string = address.to_string();
    let recovered_from_string = string.parse::<Ed25519Address>()?;
    println!("   String round trip: {}", address == recovered_from_string);

    // Encoding round trip
    let encoded = address.encode();
    let recovered_from_encoding = Ed25519Address::decode(&encoded)?;
    println!("   Encoding round trip: {}", address == recovered_from_encoding);

    println!("\n✅ All tests passed!");

    Ok(())
}
```

### Step 5: Run Tests and Examples

```bash
# Navigate to address module
cd commonware-address

# Run tests
cargo test

# Run example
cargo run --example basic_usage

# Run with verbose output
RUST_LOG=debug cargo test -- --nocapture
```

### Step 6: Integration with Workspace

Add to main `Cargo.toml`:

```toml
[workspace]
members = [
    # ... existing members ...
    "commonware-address",
]
```

Update `commonware-address` to follow Commonware conventions:

```bash
# Format code
cargo +nightly fmt

# Run lints
cargo clippy --all-targets --all-features -- -D warnings

# Test from workspace root
cd ..
cargo test -p commonware-address
```

## Next Steps

### Week 2: Enhance commonware-address

1. **Add Multi-signature Support**

   - Implement `MultiSigAddress` trait
   - Add threshold signature validation
   - Create examples for multi-sig addresses

2. **Add More Key Types**

   - Secp256k1 address support
   - BLS address support (if needed)
   - Generic address trait implementations

3. **Performance Optimization**

   - Benchmark address operations
   - Optimize hot paths
   - Add performance tests

4. **Documentation**
   - Add comprehensive rustdocs
   - Create usage tutorials
   - Document security considerations

### Validation Checklist

Before moving to Phase 2, ensure:

- [ ] All tests pass with deterministic runtime
- [ ] Examples run successfully
- [ ] Code follows Commonware style guidelines
- [ ] Documentation is complete
- [ ] Performance benchmarks exist
- [ ] Integration tests with commonware-cryptography work
- [ ] Error handling is comprehensive
- [ ] Security review completed

## Getting Help

1. **Documentation**: Check existing Commonware modules for patterns
2. **Testing**: Use `commonware-runtime/deterministic` for all async tests
3. **Performance**: Profile with `cargo bench` and optimize hot paths
4. **Style**: Follow existing Commonware conventions and run `cargo +nightly fmt`

## Common Issues

1. **Import Errors**: Ensure all dependencies are properly specified in Cargo.toml
2. **Test Failures**: Use deterministic runtime for reproducible tests
3. **Performance**: Profile before optimizing, measure improvements
4. **Integration**: Test with existing Commonware primitives early

This foundation provides a solid start for building the complete blockchain module ecosystem on Commonware.
