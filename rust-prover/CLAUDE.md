# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Rust implementation of a zero-knowledge proof (ZKP) system using Groth16 to prove age verification without revealing exact date of birth. The system uses `arkworks` libraries to implement a three-party identity verification scenario:
- **Issuer**: Issues credentials with hashed data
- **Holder**: Generates zero-knowledge proofs of age
- **Verifier**: Verifies proofs without learning sensitive information

## Commands

### Build and Run
```bash
# Run main function (requires Hardhat blockchain running and contract deployed)
cargo run --release -- --nocapture

# Run tests
cargo test --release -- --nocapture
# or
./run_test.sh

# Run specific test
cargo test --release test_did_scenario -- --nocapture
cargo test --release test_sha256_preimage_witness_only -- --nocapture
cargo test --release test_sha256_preimage_public_input -- --nocapture
```

### Prerequisites for Main Function
1. Start Hardhat in `solidity-verifier/` folder (see solidity-verifier/README.md)
2. Configure `.env` with RPC_URL and PRIVATE_KEY
3. Deploy contract using `deploy_verifier.js`
4. Update `contract_address` in main.rs:477 with deployed address

## Architecture

### Core Flow
1. **Credential Issuance** (Issuer → Holder)
   - Issuer creates credentials with: issuer_id, holder_name, holder_dob_year, randomness
   - Issuer publishes SHA256 hashes of all credentials (max: MAX_CREDENTIALS = 3)
   - These hashes act as a public registry for verification

2. **Setup** (Verifier)
   - Verifier generates proving key and verifying key using Groth16 setup
   - Uses mock circuit with dummy values to create universal keys

3. **Proof Generation** (Holder)
   - Holder constructs AgeCircuit with:
     - Public inputs: CUTOFF_YEAR (2006) and hashed_credentials from Issuer
     - Witness: holder's actual credential
   - Circuit proves TWO things:
     - Credential hash matches one of Issuer's published hashes (membership proof)
     - Birth year < CUTOFF_YEAR (age verification)
   - Generates Groth16 proof using proving key

4. **Verification** (Verifier)
   - Verifies proof locally using verifying key
   - Can verify on-chain via Solidity contract (Ethereum)

### Key Components

#### AgeCircuit (src/data_structures/circuit.rs)
R1CS circuit implementing two constraints:
- **Credential Validity**: SHA256(credential) must match one of published hashes
- **Age Check**: holder_dob_year < dob_cutoff_year (using strict comparison)

#### Credential (src/data_structures/credential.rs)
Contains: issuer_id, holder_name, holder_dob_year, randomness
Method `to_sha256()` computes hash in same order as circuit

#### Public vs Witness Data
- **Public Inputs** (visible to verifier):
  - CUTOFF_YEAR: 2006
  - hashed_credentials: 3 × 256 bits = 768 field elements (each bit becomes one F)
  - Total: 769 public inputs (1 + 768)
- **Witness** (private to prover):
  - Actual credential content (issuer_id, holder_name, holder_dob_year, randomness)

### Important Implementation Details

#### SHA256 in Circuits (src/main.rs:29-178)
- When SHA256 hash is **witness**: No public inputs needed
- When SHA256 hash is **public input**: Each of 256 bits becomes separate field element
  - Convert bytes to field elements: Little-endian bit extraction
  - Hash of 32 bytes = 256 bits = 256 field elements
  - See main.rs:434-445 for bit-to-field conversion pattern

#### Solidity Integration (src/utils/solidity/)
- `ToSolidity` trait converts arkworks types to Ethereum-compatible format
- Proof: 8 U256 values (A.x, A.y, B.x1, B.x2, B.y1, B.y2, C.x, C.y)
- Verifying Key: alpha1, beta2, gamma2, delta2, plus 770 G1 points for public inputs
- Public inputs: 769 field elements (cutoff year + 768 bits from 3 hashes)

#### Key Type Aliases (src/main.rs:18-22)
```rust
type F = ark_bn254::Fr;                    // Field for BN254 curve
type Sha256Digest = Vec<u8>;               // 32 bytes
type Groth16Proof = ...;
type Groth16ProvingKey = ...;
type Groth16VerifyingKey = ...;
```

## Analysis Structure for Learning Groth16 Identity Verification

Follow this order to understand the implementation:

### Phase 1: Understand Data Flow
1. **Read** `src/data_structures/credential.rs`
   - How credentials are structured
   - How SHA256 hash is computed (note the field order)

2. **Read** `src/entities/issuer.rs`
   - How credentials are issued
   - How hashed credentials are published
   - MAX_CREDENTIALS limit (3)

3. **Read** `src/entities/holder.rs`
   - Simple wrapper around Groth16::prove

4. **Read** `src/entities/verifier.rs`
   - How setup creates proving/verifying keys
   - Mock circuit pattern for key generation

### Phase 2: Understand Circuit Constraints
5. **Read** `src/data_structures/circuit.rs`
   - **Public inputs allocation**: Lines 27-40 (cutoff year + 3 credential hashes)
   - **Witness allocation**: Lines 44-46 (actual dob_year)
   - **Constraint 1** (lines 50-79): Hash preimage proof
     - Computes SHA256 of witness credential
     - Proves it matches one of 3 public hashed credentials (OR logic)
   - **Constraint 2** (line 82): Age comparison
     - `enforce_cmp` with `Ordering::Less` and strict=true

### Phase 3: Understand Public Input Handling
6. **Read** `src/main.rs` test `test_sha256_preimage_witness_only` (lines 104-133)
   - When hash is witness: no public inputs needed
   - Constraint count: ~27,000

7. **Read** `src/main.rs` test `test_sha256_preimage_public_input` (lines 135-178)
   - When hash is public: each bit becomes field element
   - 256 public inputs (one per bit)
   - Bit extraction pattern (lines 164-173): **Little-endian** bit order

8. **Read** `src/main.rs` test `test_did_scenario` (lines 180-282)
   - Full workflow with 3 credentials
   - Public input construction (lines 210-232): cutoff year + 3 hashes
   - Two holders: 2005 (valid) and 2007 (invalid for cutoff 2006)
   - Demonstrates selective disclosure

### Phase 4: Main Application Flow
9. **Read** `src/main.rs` main function (lines 390-489)
   - **Lines 392-417**: Issuer issues credentials
   - **Lines 422-447**: Verifier setup and public input preparation
   - **Lines 450-464**: Holder generates and verifies proof locally
   - **Lines 467-488**: Convert to Solidity format and send to blockchain

10. **Read** `src/utils/utils.rs`
    - Helper functions for byte/field conversions
    - `string_to_bytes`: converts string to Fr then to bytes

### Phase 5: Blockchain Integration (Optional)
11. **Read** `src/main.rs` function `send_tx` (lines 285-388)
    - How proof/public inputs/vk are formatted for Solidity
    - ABI interaction with Groth16Verifier contract

12. **Examine** `abi.json`
    - Solidity verifier contract interface

## Key Insights for ZKP Identity Verification

1. **Selective Disclosure**: Holder proves age >= 18 without revealing exact birth year
2. **Membership Proof**: Uses OR constraint to prove credential is in set of 3 hashed credentials
3. **Public Registry**: Issuer publishes hashes, not actual credentials
4. **Circuit Determinism**: Same hash computation in circuit (constraints) and native code (credential.rs)
5. **Bit-to-Field Mapping**: SHA256 output becomes 256 field elements for public input
6. **Setup Phase**: Mock circuit with dummy values generates universal proving/verifying keys
7. **Constraint Count**: ~27K constraints (dominated by SHA256 gadget)
