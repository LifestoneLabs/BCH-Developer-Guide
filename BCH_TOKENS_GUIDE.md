# BCH Tokens Developer Guide

A practical guide to understanding and using Bitcoin Cash tokens (CashTokens) for developers.

## Table of Contents

1. [What Are CashTokens?](#what-are-cashtokens)
2. [Token Structure](#token-structure)
3. [NFT Capabilities](#nft-capabilities)
4. [Working with Tokens in CashScript](#working-with-tokens-in-cashscript)
5. [Common Patterns](#common-patterns)
6. [Technical Considerations](#technical-considerations)

---

## What Are CashTokens?

CashTokens are Bitcoin Cash's native token protocol, enabling two types of tokens:

### Fungible Tokens
Like currencies or stablecoins - interchangeable, divisible units. Each unit is identical to every other unit.

### Non-Fungible Tokens (NFTs)
Unique, indivisible tokens representing specific items like art, tickets, or credentials. Each NFT is distinct.

### The UTXO Model

Unlike Ethereum's account-based system, CashTokens use Bitcoin's UTXO (Unspent Transaction Output) model:

**Account-Based (Ethereum):**
```
Alice's Account: 100 tokens
Bob's Account: 50 tokens
Transfer 10: Alice -= 10, Bob += 10
```

**UTXO-Based (Bitcoin Cash):**
```
UTXO #1: 100 tokens → Alice
UTXO #2: 50 tokens → Bob
Transfer 10: 
  - Create UTXO #3: 10 tokens → Bob
  - Create UTXO #4: 90 tokens → Alice
```

**Key implications:**
- Each token UTXO is a separate entity
- To spend tokens, you spend the entire UTXO and create new ones
- No global token balance - just individual UTXOs
- Parallel validation is possible

---

## Token Structure

A Bitcoin Cash UTXO (Unspent Transaction Output) has the following structure:

```
UTXO
├── satoshis: int              // Amount of BCH (in satoshis)
├── txid: bytes32              // Transaction ID this output belongs to
├── vout: int                  // Output index (0, 1, 2...)
└── token: Token (optional)    // Token data (may be empty)
    ├── amount: int            // Fungible token amount (0 for pure NFTs)
    ├── category: bytes32      // Unique token identifier (genesis txid)
    └── nft: NFT (optional)    // NFT-specific data
        ├── capability: bytes1 // 0x00=none, 0x01=mutable, 0x02=minting
        └── commitment: bytes  // 0-40 bytes of arbitrary data
```

### Example UTXO with Token:

```
UTXO
├── satoshis: 1000
├── txid: a3f2c8d9...
├── vout: 0
└── token
    ├── amount: 5000
    ├── category: b8e1a4c2...
    └── nft
        ├── capability: 0x00 (none)
        └── commitment: 0x01a4
```

### Example Pure BCH UTXO (no token):

```
UTXO
├── satoshis: 10000
├── txid: d7e5b3a1...
├── vout: 1
└── token: null
```

Every token output contains:

### Token Category (32 bytes)
The unique identifier for a token type. Typically the transaction ID of the genesis transaction that created the token.

```
Genesis Transaction TXID: abc123...
└─ Becomes the category ID for all tokens of this type
```

### Token Amount
For fungible tokens: the number of tokens (can be very large).
For pure NFTs: always 0.

### NFT Data (optional)
If the output contains an NFT:

**Capability** (1 byte):
- `0x00` (none): Standard NFT, immutable
- `0x01` (mutable): Can update commitment data
- `0x02` (minting): Can create more tokens of this category

**Commitment** (0-40 bytes):
Arbitrary data stored with the NFT - serial numbers, hashes, metadata references, etc.

---

## NFT Capabilities

### None (0x00)
Standard immutable NFT. Once created, its commitment cannot change.

**Use cases:** Digital art, collectibles, proof-of-ownership documents.

### Mutable (0x01)
NFT that can update its commitment over time.

**Use cases:** Gaming items that level up, dynamic NFTs, status credentials.

**Important:** Only the commitment can change - the NFT itself must be spent and recreated to "update" it. The new output must also have `mutable` capability.

### Minting (0x02)
Authority NFT that can create new tokens of the same category.

**Use cases:** Token issuers, DAO treasuries, game masters spawning items.

**Important:** The minting NFT is not consumed when minting - it must be preserved in an output to maintain the authority.

---

## Working with Tokens in CashScript

### Reading Token Data from Inputs

```cashscript
pragma cashscript ^0.12.0;

contract TokenReader() {
    function readToken() {
        // Get the full token category bytes (33 bytes: 32 category + 1 capability)
        bytes tokenData = tx.inputs[0].tokenCategory;
        
        // Split to get category ID and capability separately
        bytes32 categoryId = tokenData.split(32)[0];
        bytes1 capability = tokenData.split(32)[1];
        
        // Read fungible amount
        int amount = tx.inputs[0].tokenAmount;
        
        // Read NFT commitment
        bytes commitment = tx.inputs[0].nftCommitment;
        
        // Verify it's a minting NFT
        require(capability == 0x02);
        
        // Verify specific commitment
        require(commitment == 0x00c9);
    }
}
```

### Carrying Tokens Forward (Input → Output)

The most common pattern is preserving tokens from input to output:

```cashscript
pragma cashscript ^0.12.0;

contract TokenTransfer() {
    function transfer() {
        // Read token from input
        bytes32 category = tx.inputs[0].tokenCategory.split(32)[0];
        int amount = tx.inputs[0].tokenAmount;
        bytes commitment = tx.inputs[0].nftCommitment;
        bytes1 capability = tx.inputs[0].tokenCategory.split(32)[1];
        
        // Send to recipient - preserving all token properties
        require(tx.outputs[0].tokenCategory.split(32)[0] == category);
        require(tx.outputs[0].tokenAmount == amount);
        require(tx.outputs[0].nftCommitment == commitment);
        require(tx.outputs[0].tokenCategory.split(32)[1] == capability);
        
        // Also verify the satoshi value is preserved
        require(tx.outputs[0].value == tx.inputs[0].value);
    }
}
```

### Checking for Token Presence

```cashscript
// Check if input has any token
bool hasToken = tx.inputs[0].tokenCategory != 0x;

// Check if it's specifically an NFT
bool isNFT = tx.inputs[0].nftCommitment != 0x || 
             tx.inputs[0].tokenCategory.split(32)[1] != 0x00;
```

---

## Common Patterns

### 1. Simple NFT Transfer

```cashscript
pragma cashscript ^0.12.0;

contract NFTTransfer(
    bytes recipientLockingBytecode
) {
    function transfer() {
        // Verify input has NFT
        require(tx.inputs[0].tokenCategory != 0x);
        
        // Get NFT data
        bytes tokenData = tx.inputs[0].tokenCategory;
        bytes32 category = tokenData.split(32)[0];
        bytes commitment = tx.inputs[0].nftCommitment;
        
        // Output must preserve NFT to recipient
        require(tx.outputs[0].tokenCategory.split(32)[0] == category);
        require(tx.outputs[0].nftCommitment == commitment);
        require(tx.outputs[0].lockingBytecode == recipientLockingBytecode);
        require(tx.outputs[0].value >= 546); // Dust limit
    }
}
```

### 2. Fungible Token Split

```cashscript
pragma cashscript ^0.12.0;

contract TokenSplitter() {
    function split(
        int amountToSend,
        bytes recipientLockingBytecode
    ) {
        // Get input token data
        bytes32 category = tx.inputs[0].tokenCategory.split(32)[0];
        int totalAmount = tx.inputs[0].tokenAmount;
        
        // Verify sufficient balance
        require(totalAmount >= amountToSend);
        int change = totalAmount - amountToSend;
        
        // Send to recipient
        require(tx.outputs[0].tokenCategory.split(32)[0] == category);
        require(tx.outputs[0].tokenAmount == amountToSend);
        require(tx.outputs[0].lockingBytecode == recipientLockingBytecode);
        
        // Return change (if any)
        if (change > 0) {
            require(tx.outputs[1].tokenCategory.split(32)[0] == category);
            require(tx.outputs[1].tokenAmount == change);
        }
    }
}
```

### 3. Mutable NFT Update

```cashscript
pragma cashscript ^0.12.0;

contract EvolvingNFT(
    bytes32 expectedCategory
) {
    function update(bytes newCommitment) {
        // Verify correct category
        bytes32 category = tx.inputs[0].tokenCategory.split(32)[0];
        require(category == expectedCategory);
        
        // Verify mutable capability
        bytes1 capability = tx.inputs[0].tokenCategory.split(32)[1];
        require(capability == 0x01);
        
        // Output must keep mutable capability with new commitment
        require(tx.outputs[0].tokenCategory.split(32)[0] == category);
        require(tx.outputs[0].tokenCategory.split(32)[1] == 0x01);
        require(tx.outputs[0].nftCommitment == newCommitment);
        require(tx.outputs[0].tokenAmount == 0); // Still pure NFT
    }
}
```

### 4. Minting with Authority NFT

```cashscript
pragma cashscript ^0.12.0;

contract TokenMinter(
    bytes32 categoryId
) {
    function mint(
        int mintAmount,
        bytes recipientLockingBytecode
    ) {
        // Verify input has minting capability
        bytes32 inputCategory = tx.inputs[0].tokenCategory.split(32)[0];
        bytes1 capability = tx.inputs[0].tokenCategory.split(32)[1];
        
        require(inputCategory == categoryId);
        require(capability == 0x02);
        
        // Must preserve minting NFT
        require(tx.outputs[0].tokenCategory == tx.inputs[0].tokenCategory);
        require(tx.outputs[0].nftCommitment == tx.inputs[0].nftCommitment);
        require(tx.outputs[0].tokenAmount == 0);
        
        // Newly minted tokens go to recipient
        require(tx.outputs[1].tokenCategory.split(32)[0] == categoryId);
        require(tx.outputs[1].tokenAmount == mintAmount);
        require(tx.outputs[1].lockingBytecode == recipientLockingBytecode);
    }
}
```

---

## Technical Considerations

### Dust Limits

Outputs must have at least 546 satoshis to be spendable:

```cashscript
require(tx.outputs[0].value >= 546);
```

This applies to token outputs too - they need both the token AND sufficient BCH.

### Token Burns

If you don't include a token in any output, it gets burned (destroyed):

```cashscript
// BAD: Input has token but output[0] doesn't include it
// This burns the token!

// GOOD: Explicitly preserve or handle the token
require(tx.outputs[0].tokenCategory == tx.inputs[0].tokenCategory);
```

**Always verify token preservation** unless intentional burning is the goal.

### Genesis Transactions

To create a new token category:

1. Spend a UTXO that will become vout 0
2. The transaction ID of this spend becomes the category ID
3. vout 0 becomes the "authhead" - the controlling UTXO for BCMR metadata

```
Input: Some BCH UTXO
│
└─► Transaction (becomes new category ID)
    ├─ Output 0: Authhead (vout 0 - controls category)
    │   └─ Token: {category: <this_tx_id>, amount: 0, nft: {...}}
    └─ Output 1+: Other outputs
```

### Commitment Size

NFT commitments are limited to 40 bytes. For larger data, store a hash in the commitment and the full data elsewhere (IPFS, BCMR, etc.):

```cashscript
// Store 32-byte hash (fits in commitment)
commitment: 0xa1b2c3d4... // SHA256 hash of actual data
```

### Capability Preservation

When spending an NFT, you must explicitly set the capability in the output:

```cashscript
// Preserving minting capability
require(tx.outputs[0].tokenCategory.split(32)[1] == 0x02);

// Downgrading from minting to none (one-way)
require(tx.outputs[0].tokenCategory.split(32)[1] == 0x00);
```

### Testing

Always test token contracts on chipnet before mainnet:

```bash
# Compile for chipnet
cashc mytoken.cash --network chipnet

# Get chipnet BCH from faucet
curl https://chipnet.imaginary.cash/faucet
```

### BCMR and Authheads

The authhead (vout 0 of genesis) controls BCMR metadata:

- To update token metadata, you must spend the authhead
- The new authhead becomes vout 0 of the spending transaction
- Best practice: Move authhead to a dedicated address you control

```
Genesis TX (category = this txid)
├─ vout 0: Authhead ──► (spend to update BCMR)
│                       └─ New TX
│                           └─ vout 0: New Authhead
└─ vout 1: First token minted
```

### Common Mistakes

1. **Forgetting to preserve the minting NFT**: When minting, the authority NFT must be output unchanged.

2. **Not checking capability**: Always verify `tokenCategory.split(32)[1]` matches expected capability.

3. **Burning tokens accidentally**: Always verify token outputs match inputs unless burning is intentional.

4. **Wrong commitment format**: Commitments are raw bytes, not strings. Hex encode if needed.

5. **Ignoring dust limit**: Token outputs must have ≥546 satoshis.

---

## Resources

- [CashScript Documentation](https://cashscript.org/)
- [CashTokens CHIP](https://github.com/bitjson/cashtokens)
- [BCMR Protocol](https://cashtokens.org/bcmr.html)
- [Chipnet Faucet](https://chipnet.imaginary.cash/faucet)

---

*This guide is for CashScript 0.12.0.*
