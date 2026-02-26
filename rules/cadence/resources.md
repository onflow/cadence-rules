# Cadence Resource Rules

> **Purpose**: Define comprehensive rules for working with resources in Cadence. Resources are the foundation of Cadence's security model and represent unique, owned assets that cannot be copied or lost.

## Core Philosophy: Linear Typing and Resource Safety

Resources in Cadence follow **linear typing** principles: they exist in exactly one location at a time and must be explicitly handled. This prevents duplication, loss, and unauthorized access.

## What Are Resources?

Resources are special types marked with the `@` symbol that represent **valuable, unique assets**:

- **Digital assets**: NFTs, tokens, certificates
- **Access rights**: Capabilities, permissions, keys
- **Unique state**: User accounts, game characters, licenses

## Three Fundamental Rules

### Rule 1: Singular Existence
**Resources can only exist in ONE location at a time.**

```cadence
// ✅ CORRECT: Resource moves from one location to another
let nft <- collection.withdraw(id: 1)
otherCollection.deposit(token: <-nft)  // Now in new location

// ❌ IMPOSSIBLE: Cannot copy or duplicate
let nft <- collection.withdraw(id: 1)
let copy = nft  // COMPILE ERROR: Missing move operator
```

### Rule 2: Mandatory Handling
**Resources must be explicitly moved or destroyed.**

```cadence
// ✅ CORRECT: Explicit handling
let vault <- createVault()
destroy vault  // Explicitly destroyed

// ✅ CORRECT: Moved to storage
let vault <- createVault()
account.storage.save(<-vault, to: /storage/vault)

// ❌ WRONG: Resource would be lost
let vault <- createVault()
// Function ends without handling vault - COMPILE ERROR
```

### Rule 3: Explicit Syntax
**Use `@` for types and `<-` for operations.**

```cadence
// ✅ CORRECT: Explicit resource syntax
let vault: @Vault <- createVault()
account.storage.save(<-vault, to: /storage/vault)

// ❌ WRONG: Missing resource syntax
let vault: Vault = createVault()  // COMPILE ERROR
account.storage.save(vault, to: /storage/vault)  // COMPILE ERROR
```

## Resource Declaration

### Basic Resource Declaration

```cadence
access(all) resource NFT {
    access(all) let id: UInt64
    access(all) let metadata: {String: String}

    init(id: UInt64, metadata: {String: String}) {
        self.id = id
        self.metadata = metadata
    }
}
```

### Resource with Interfaces

```cadence
access(all) resource interface NFTPublic {
    access(all) let id: UInt64
    access(all) fun getMetadata(): {String: String}
}

access(all) resource NFT: NFTPublic {
    access(all) let id: UInt64
    access(self) var metadata: {String: String}

    access(all) fun getMetadata(): {String: String} {
        return self.metadata
    }

    init(id: UInt64, metadata: {String: String}) {
        self.id = id
        self.metadata = metadata
    }
}
```

## Resource Creation

### Using the `create` Keyword

**Resources are created with the `create` keyword**:

```cadence
// ✅ CORRECT: Create and immediately move
let nft <- create NFT(id: 1, metadata: {})

// ✅ CORRECT: Create in function
access(all) fun mintNFT(id: UInt64): @NFT {
    return <- create NFT(id: id, metadata: {})
}

// ❌ WRONG: Missing move operator
let nft = create NFT(id: 1, metadata: {})  // COMPILE ERROR
```

### Creation in Contract Functions

```cadence
access(all) contract NFTContract {
    access(all) resource NFT {
        access(all) let id: UInt64

        init(id: UInt64) {
            self.id = id
        }
    }

    // Minting function
    access(all) fun mintNFT(id: UInt64): @NFT {
        return <- create NFT(id: id)
    }
}
```

## The Move Operator (`<-`)

### When to Use `<-`

**Always use `<-` when working with resources**:

1. **Assignment**
```cadence
let vault <- createVault()
```

2. **Function Arguments**
```cadence
fun deposit(vault: @Vault) {
    // Implementation
}

deposit(vault: <-myVault)
```

3. **Function Returns**
```cadence
fun withdraw(): @Vault {
    return <- create Vault()
}
```

4. **Storage Operations**
```cadence
// Save
account.storage.save(<-vault, to: /storage/vault)

// Load
let vault <- account.storage.load<@Vault>(from: /storage/vault)
```

5. **Array/Dictionary Operations**
```cadence
// Add to array (special syntax)
vaults.append(<-vault)

// Add to dictionary (special syntax)
nfts[id] <-! nft
```

### Move Operator Rules

```cadence
// ✅ CORRECT: Explicit moves
let nft1 <- create NFT(id: 1)
let nft2 <- nft1  // nft1 is now invalid
collection.deposit(<-nft2)

// ❌ WRONG: Using after move
let nft <- create NFT(id: 1)
collection.deposit(<-nft)
log(nft.id)  // COMPILE ERROR: nft no longer valid

// ✅ CORRECT: Check before using
let nft <- create NFT(id: 1)
let id = nft.id  // Read before moving
collection.deposit(<-nft)
log(id)  // Safe: using copied value
```

## Resource Destruction

### Explicit Destruction

**Resources must be explicitly destroyed when no longer needed**:

```cadence
// ✅ CORRECT: Explicit destroy
let vault <- createVault()
destroy vault

// ✅ CORRECT: Conditional destruction
let vault <- createVault()
if shouldKeep {
    account.storage.save(<-vault, to: /storage/vault)
} else {
    destroy vault
}
```

### Destroy Events

Resources can emit events when destroyed:

```cadence
access(all) resource NFT {
    access(all) let id: UInt64

    init(id: UInt64) {
        self.id = id
    }

    // Emitted automatically when destroyed
    destroy() {
        emit NFTDestroyed(id: self.id)
    }
}

access(all) event NFTDestroyed(id: UInt64)
```

### Nested Resource Destruction

**Parent destruction automatically destroys nested resources**:

```cadence
access(all) resource Collection {
    access(self) var nfts: @{UInt64: NFT}

    init() {
        self.nfts <- {}
    }

    destroy() {
        // All NFTs in the collection are destroyed automatically
        destroy self.nfts
    }
}

// When you destroy a collection, all its NFTs are destroyed too
let collection <- createCollection()
destroy collection  // Destroys collection AND all NFTs inside
```

## Resource Arrays and Dictionaries

### Special Operators for Collections

**Resources in arrays/dictionaries require special operators**:

#### Swap Operator (`<->`)

```cadence
access(all) resource Collection {
    access(self) var nfts: @{UInt64: NFT}

    // Swap resources in dictionary
    access(all) fun swapNFT(id: UInt64, new: @NFT): @NFT {
        var old <- self.nfts[id] <- new
        return <-old
    }
}
```

#### Shift Operator (`<- target <-`)

```cadence
// Replace and get old value in one operation
let old <- collection[id] <- new
```

### Working with Resource Arrays

```cadence
access(all) resource Vault {
    access(self) var balance: UFix64
    init(balance: UFix64) {
        self.balance = balance
    }
}

access(all) resource VaultCollection {
    access(self) var vaults: @[Vault]

    init() {
        self.vaults <- []
    }

    // Add vault to array
    access(all) fun addVault(vault: @Vault) {
        self.vaults.append(<-vault)
    }

    // Remove vault from array
    access(all) fun removeVault(index: Int): @Vault {
        return <- self.vaults.remove(at: index)
    }

    // Cannot directly access: self.vaults[0]
    // Must remove, use, and optionally re-insert

    destroy() {
        destroy self.vaults
    }
}
```

### Working with Resource Dictionaries

```cadence
access(all) resource NFTCollection {
    access(self) var nfts: @{UInt64: NFT}

    init() {
        self.nfts <- {}
    }

    // Deposit (insert or replace)
    access(all) fun deposit(nft: @NFT) {
        let id = nft.id
        let old <- self.nfts[id] <- nft
        destroy old  // Destroy if there was an old one
    }

    // Withdraw (remove and return)
    access(all) fun withdraw(id: UInt64): @NFT {
        let nft <- self.nfts.remove(key: id)
            ?? panic("NFT with ID \(id) not found in collection")
        return <-nft
    }

    // Safe deposit (don't overwrite)
    access(all) fun safeDeposit(nft: @NFT) {
        let id = nft.id

        // Check if already exists
        assert(self.nfts[id] == nil, message: "NFT with ID \(id) already exists in collection")

        // Insert (use force-insert operator <-!)
        self.nfts[id] <-! nft
    }

    destroy() {
        destroy self.nfts
    }
}
```

## Resource Scope and Validity

### Valid Resource Usage

```cadence
// ✅ CORRECT: All paths handle resource
fun processVault(vault: @Vault): Bool {
    if vault.balance > 100.0 {
        account.storage.save(<-vault, to: /storage/bigVault)
        return true
    } else {
        destroy vault
        return false
    }
    // Both paths handle the resource
}

// ❌ WRONG: Not all paths handle resource
fun processVault(vault: @Vault): Bool {
    if vault.balance > 100.0 {
        account.storage.save(<-vault, to: /storage/bigVault)
        return true
    }
    return false  // COMPILE ERROR: vault not handled in else path
}
```

### Resource Invalidation

**Once moved, a resource variable becomes invalid**:

```cadence
// ✅ CORRECT: Use before moving
let nft <- create NFT(id: 1)
log(nft.id)  // Read value
collection.deposit(<-nft)  // Move resource

// ❌ WRONG: Use after moving
let nft <- create NFT(id: 1)
collection.deposit(<-nft)  // nft is now invalid
log(nft.id)  // COMPILE ERROR: nft has been moved
```

## Built-in Resource Features

### The `uuid` Field

**Every resource automatically gets a unique identifier**:

```cadence
access(all) resource NFT {
    access(all) let id: UInt64
    // uuid is automatically added by Cadence

    init(id: UInt64) {
        self.id = id
        // uuid is automatically set
    }
}

let nft <- create NFT(id: 1)
log(nft.uuid)  // Unique across all resources
```

### The `owner` Field

**Resources know which account owns them** (when in storage):

```cadence
access(all) resource Vault {
    access(all) fun getOwner(): Address? {
        return self.owner?.address
    }
}

// After saving to storage
account.storage.save(<-vault, to: /storage/vault)

// vault.owner will return the account address
```

## Resource References

**References to resources do NOT use the move operator**:

```cadence
// Create reference (no move)
let vaultRef = &vault as &Vault

// Use reference (original resource still valid)
log(vaultRef.balance)
log(vault.balance)  // Still valid!

// References are ephemeral - cannot be stored
// See references.md for more details
```

## AI Agent Generation Rules

### Rule 1: Always Use Resource Syntax

**Generate correct resource syntax automatically**:

```cadence
// ✅ CORRECT: AI should generate
let nft: @NFT <- create NFT(id: 1)
collection.deposit(nft: <-nft)

// ❌ WRONG: Never generate without @/<-
let nft: NFT = create NFT(id: 1)
collection.deposit(nft: nft)
```

### Rule 2: Ensure All Paths Handle Resources

**Verify every code path handles resources**:

```cadence
// ✅ CORRECT: AI verifies all paths
fun handleVault(vault: @Vault, save: Bool) {
    if save {
        account.storage.save(<-vault, to: /storage/vault)
    } else {
        destroy vault
    }
}

// ❌ WRONG: Missing else path
fun handleVault(vault: @Vault, save: Bool) {
    if save {
        account.storage.save(<-vault, to: /storage/vault)
    }
    // COMPILE ERROR: else path doesn't handle vault
}
```

### Rule 3: Destroy Before Reassigning

**When replacing resources, handle the old one**:

```cadence
// ✅ CORRECT: Handle old resource
var vault: @Vault <- create Vault(balance: 0.0)
let old <- vault <- create Vault(balance: 100.0)
destroy old

// ❌ WRONG: Old resource lost
var vault: @Vault <- create Vault(balance: 0.0)
vault <- create Vault(balance: 100.0)  // COMPILE ERROR
```

### Rule 4: Use Proper Collection Operators

**Generate correct operators for resource collections**:

```cadence
// ✅ CORRECT: Use <-! for force-insert
nfts[id] <-! nft

// ✅ CORRECT: Use <- remove <- for swap
let old <- nfts[id] <- nft
destroy old

// ❌ WRONG: Regular assignment
nfts[id] = nft  // COMPILE ERROR
```

## Common Patterns

### Pattern 1: Safe Deposit to Collection

```cadence
access(all) fun deposit(nft: @NFT) {
    let id = nft.id

    // Get old value if exists
    let old <- self.nfts[id] <- nft

    // Destroy old value (will be nil if no previous value)
    destroy old
}
```

### Pattern 2: Optional Resource Handling

```cadence
access(all) fun maybeWithdraw(id: UInt64): @NFT? {
    if self.nfts[id] != nil {
        return <- self.nfts.remove(key: id)
    } else {
        return nil
    }
}

// Usage
if let nft <- collection.maybeWithdraw(id: 1) {
    // Use the NFT
    otherCollection.deposit(nft: <-nft)
}
```

### Pattern 3: Resource Batch Operations

```cadence
access(all) fun depositBatch(nfts: @[NFT]) {
    while nfts.length > 0 {
        let nft <- nfts.removeFirst()
        self.deposit(nft: <-nft)
    }
    destroy nfts  // Destroy empty array
}
```

### Pattern 4: Resource Transformation

```cadence
access(all) resource OldNFT {
    access(all) let id: UInt64
    init(id: UInt64) { self.id = id }
}

access(all) resource NewNFT {
    access(all) let id: UInt64
    access(all) let upgraded: Bool
    init(id: UInt64) {
        self.id = id
        self.upgraded = true
    }
}

access(all) fun upgrade(old: @OldNFT): @NewNFT {
    let id = old.id  // Copy data before destroying
    destroy old      // Destroy old resource
    return <- create NewNFT(id: id)  // Create new resource
}
```

## Security Considerations

### Resource Ownership

**Resources in storage are owned by the account**:

```cadence
// Only the account can access stored resources
let vault <- account.storage.load<@Vault>(from: /storage/vault)

// Others can only borrow references (if capability exists)
let vaultRef = capability.borrow<&Vault>()
```

### Preventing Resource Loss

**Always ensure resources are handled**:

```cadence
// ✅ CORRECT: Transaction handles all resources
transaction() {
    prepare(signer: auth(Storage) &Account) {
        let vault <- signer.storage.load<@Vault>(from: /storage/oldVault)
            ?? panic("Vault not found at /storage/oldVault")

        signer.storage.save(<-vault, to: /storage/newVault)
    }
}

// ❌ WRONG: Resource could be lost
transaction() {
    prepare(signer: auth(Storage) &Account) {
        let vault <- signer.storage.load<@Vault>(from: /storage/oldVault)
        // If panic happens here, vault is lost!
        someOperation()
    }
}
```

## Testing Resources

```cadence
import Test

access(all) fun testResourceCreation() {
    let nft <- create NFT(id: 1)

    Test.expect(nft.id, Test.equal(1))
    Test.expect(nft.uuid, Test.not(Test.equal(0)))

    destroy nft
}

access(all) fun testResourceMovement() {
    let collection <- create Collection()
    let nft <- create NFT(id: 1)

    collection.deposit(nft: <-nft)

    Test.expect(collection.getIDs(), Test.equal([1]))

    destroy collection
}
```

## Reference Links

- [Cadence Resources Documentation](https://cadence-lang.org/docs/language/resources)
- [Resource-Oriented Programming](https://www.onflow.org/post/resource-oriented-programming)
- [Linear Types in Cadence](https://www.onflow.org/post/flow-blockchain-cadence-programming-language-resources-assets)
