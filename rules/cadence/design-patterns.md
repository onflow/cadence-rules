# Cadence Design Patterns

> **Purpose**: Comprehensive collection of proven design patterns for Cadence development. These patterns promote maintainability, security, and efficiency.

## General Patterns

### Pattern 1: Named Constants

**Problem**: Magic numbers and hardcoded values scattered throughout code.

**Solution**: Define public constant fields in the responsible contract.

```cadence
// ❌ ANTI-PATTERN: Magic numbers
access(all) contract BadContract {
    access(all) fun calculateFee(amount: UFix64): UFix64 {
        return amount * 0.05  // What is 0.05?
    }

    access(all) fun isValidAmount(amount: UFix64): Bool {
        return amount >= 10.0 && amount <= 1000.0  // Magic numbers
    }
}

// ✅ PATTERN: Named constants
access(all) contract GoodContract {
    // Public constants
    access(all) let FEE_PERCENTAGE: UFix64
    access(all) let MINIMUM_AMOUNT: UFix64
    access(all) let MAXIMUM_AMOUNT: UFix64

    access(all) fun calculateFee(amount: UFix64): UFix64 {
        return amount * self.FEE_PERCENTAGE
    }

    access(all) fun isValidAmount(amount: UFix64): Bool {
        return amount >= self.MINIMUM_AMOUNT && amount <= self.MAXIMUM_AMOUNT
    }

    init() {
        self.FEE_PERCENTAGE = 0.05
        self.MINIMUM_AMOUNT = 10.0
        self.MAXIMUM_AMOUNT = 1000.0
    }
}
```

**Benefits**:
- Single source of truth
- Easy to update values
- Self-documenting code
- Consistent values across transactions

### Pattern 2: Script-Accessible Public Data

**Problem**: Need to query contract data without transaction fees.

**Solution**: Expose data through public fields and functions.

```cadence
access(all) contract TokenContract {
    // Public constant - scripts can read
    access(all) let totalSupply: UFix64

    // Public view function - scripts can call
    access(all) view fun getTotalSupply(): UFix64 {
        return self.totalSupply
    }

    // Public view function with computation
    access(all) view fun getCirculatingSupply(): UFix64 {
        return self.totalSupply - self.burnedSupply
    }

    access(self) var burnedSupply: UFix64

    init() {
        self.totalSupply = 1000000.0
        self.burnedSupply = 0.0
    }
}
```

**Script usage**:
```cadence
import "TokenContract"

access(all) fun main(): UFix64 {
    return TokenContract.getTotalSupply()
}
```

**Benefits**:
- No transaction fees for queries
- Fast data access
- Real-time information

### Pattern 3: Report Structs for Resources

**Problem**: Scripts cannot return resources to external context.

**Solution**: Create struct reports that mirror resource data.

```cadence
access(all) contract NFTContract {
    // Report struct (can be returned by scripts)
    access(all) struct NFTReport {
        access(all) let id: UInt64
        access(all) let owner: Address
        access(all) let metadata: {String: String}

        init(id: UInt64, owner: Address, metadata: {String: String}) {
            self.id = id
            self.owner = owner
            self.metadata = metadata
        }
    }

    // Resource (cannot be returned by scripts)
    access(all) resource NFT {
        access(all) let id: UInt64
        access(self) let metadata: {String: String}

        // Generate report from resource
        access(all) fun generateReport(): NFTReport {
            return NFTReport(
                id: self.id,
                owner: self.owner?.address ?? panic("No owner"),
                metadata: self.metadata
            )
        }

        init(id: UInt64, metadata: {String: String}) {
            self.id = id
            self.metadata = metadata
        }
    }
}
```

**Script usage**:
```cadence
import "NFTContract"

access(all) fun main(owner: Address, nftID: UInt64): NFTContract.NFTReport {
    let collection = getAccount(owner)
        .capabilities.borrow<&Collection>(/public/collection)
        ?? panic("Could not borrow Collection reference from /public/collection")

    let nft = collection.borrowNFT(id: nftID)
        ?? panic("NFT with ID \(nftID) not found in collection")

    return nft.generateReport()  // Returns struct, not resource
}
```

**Benefits**:
- Scripts can query resource data
- Type-safe data transfer
- Structured responses

### Pattern 4: Singleton Admin Resource

**Problem**: Need single admin resource created only at deployment.

**Solution**: Create admin in contract `init()` function.

```cadence
access(all) contract SecureContract {
    access(all) resource Admin {
        access(all) fun performAdminAction() {
            // Privileged operation
        }

        // Admin can create more admins if needed
        access(all) fun createAdmin(): @Admin {
            return <- create Admin()
        }
    }

    init() {
        // Create single admin instance
        let admin <- create Admin()

        // Save to deployer's account
        self.account.storage.save(<-admin, to: /storage/contractAdmin)

        // Log creation
        log("Admin created and saved")
    }

    // NO public creation function!
}
```

**Benefits**:
- Single point of admin creation
- Cannot be called by unauthorized accounts
- Secure by design

### Pattern 5: Descriptive Naming

**Problem**: Unclear variable and path names reduce code readability.

**Solution**: Use descriptive, specific names.

```cadence
// ❌ ANTI-PATTERN: Unclear names
transaction(pcnt: UFix64, addr: Address) {
    let acct1: &Account
    let acct2: &Account
    let c: @Collection

    prepare(a: auth(BorrowValue) &Account) {
        self.acct1 = a
        // ...
    }
}

// ✅ PATTERN: Descriptive names
transaction(taxPercentage: UFix64, recipientAddress: Address) {
    let payerAccount: &Account
    let recipientAccount: &Account
    let nftCollection: @NFTCollection

    prepare(payer: auth(BorrowValue) &Account) {
        self.payerAccount = payer
        // ...
    }
}
```

**Naming guidelines**:
- Use full words, not abbreviations
- Indicate purpose, not just type
- Be specific about what the variable represents
- Use camelCase for multi-word names

### Pattern 6: Plural Names for Collections

**Problem**: Unclear whether variable holds single item or collection.

**Solution**: Use plural names for arrays and dictionaries.

```cadence
// ❌ CONFUSING: Singular for collection
access(all) contract TokenContract {
    access(self) var account: [Address]  // Actually multiple accounts!

    access(all) fun processAccount() {
        for a in self.account {  // "account" is misleading
            // Process each account
        }
    }
}

// ✅ CLEAR: Plural for collection
access(all) contract TokenContract {
    access(self) var accounts: [Address]

    access(all) fun processAccounts() {
        for account in self.accounts {  // Clear: iterating accounts
            // Process each account
        }
    }
}
```

**Benefits**:
- Immediate clarity about data structure
- Natural iteration variable names
- Reduced cognitive load

### Pattern 7: Transaction Post-Conditions

**Problem**: Need to verify transaction outcomes.

**Solution**: Use post-conditions to document and enforce expected results.

```cadence
transaction(nftID: UInt64, price: UFix64) {
    let buyerCollection: &NFTCollection
    let sellerCollection: &NFTCollection
    let payment: @Vault

    prepare(buyer: auth(BorrowValue, SaveValue) &Account) {
        self.buyerCollection = buyer.storage
            .borrow<&NFTCollection>(from: /storage/collection)
            ?? panic("Could not borrow NFTCollection reference from /storage/collection")

        // Create payment
        let vault = buyer.storage
            .borrow<auth(FungibleToken.Withdraw) &Vault>(from: /storage/vault)
            ?? panic("Could not borrow Vault reference from /storage/vault")

        self.payment <- vault.withdraw(amount: price)

        // Borrow seller collection
        self.sellerCollection = getAccount(sellerAddress)
            .capabilities.borrow<&NFTCollection>(/public/collection)
            ?? panic("Could not borrow NFTCollection reference from /public/collection")
    }

    execute {
        // Purchase NFT
        let nft <- self.sellerCollection.withdraw(id: nftID)
        self.buyerCollection.deposit(nft: <-nft)

        // Pay seller
        self.sellerCollection.depositPayment(from: <-self.payment)
    }

    post {
        self.buyerCollection.owns(nftID): "Buyer does not own NFT after purchase"
        !self.sellerCollection.owns(nftID): "Seller still owns NFT after sale"
    }
}
```

**Benefits**:
- Verifies expected outcomes
- Documents transaction intent
- Catches unexpected behavior

### Pattern 8: Borrow Instead of Load/Save

**Problem**: Loading and saving resources is expensive and unnecessary.

**Solution**: Use `borrow()` for in-place mutations.

```cadence
// ❌ INEFFICIENT: Load and save
transaction() {
    prepare(signer: auth(LoadValue, SaveValue) &Account) {
        // Load resource (expensive)
        let vault <- signer.storage.load<@Vault>(from: /storage/vault)
            ?? panic("Vault not found at /storage/vault")

        // Modify
        vault.someFunction()

        // Save back (expensive)
        signer.storage.save(<-vault, to: /storage/vault)
    }
}

// ✅ EFFICIENT: Borrow reference
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Borrow reference (cheap)
        let vaultRef = signer.storage
            .borrow<&Vault>(from: /storage/vault)
            ?? panic("Could not borrow Vault reference from /storage/vault")

        // Modify in place
        vaultRef.someFunction()

        // No save needed - resource stays in storage
    }
}
```

**Benefits**:
- Dramatically reduced gas costs
- Faster execution
- Simpler code
- Resource never leaves storage

## Capability Patterns

### Pattern 9: Capability Bootstrapping via Inbox

**Problem**: Setting up cross-account capabilities requires coordination.

**Solution**: Use inbox API for asynchronous capability publishing and claiming.

```cadence
// Step 1: Provider publishes capability
transaction(recipientAddress: Address) {
    prepare(provider: auth(IssueStorageCapabilityController, Inbox) &Account) {
        // Create capability
        let controller = provider.capabilities.storage
            .issue<&MyResource>(/storage/myResource)

        controller.setTag("Access to MyResource")

        // Publish to recipient's inbox
        provider.inbox.publish(
            controller.capability,
            name: "myResourceAccess",
            recipient: recipientAddress
        )
    }
}

// Step 2: Recipient claims capability
transaction(providerAddress: Address) {
    prepare(recipient: auth(Inbox, SaveValue) &Account) {
        // Claim from inbox
        let cap = recipient.inbox.claim<&MyResource>(
            name: "myResourceAccess",
            provider: providerAddress
        ) ?? panic("Capability named \"myResourceAccess\" not found in inbox from provider \(providerAddress)")

        // Save or use capability
        recipient.storage.save(cap, to: /storage/receivedCapability)
    }
}
```

**Benefits**:
- No coordination needed
- Asynchronous setup
- Provider controls capability
- Can be done in separate transactions

### Pattern 10: Pre-Issuance Capability Checks

**Problem**: Duplicate capabilities waste storage and cause confusion.

**Solution**: Check for existing capabilities before issuing new ones.

```cadence
transaction() {
    prepare(signer: auth(IssueStorageCapabilityController, PublishCapability) &Account) {
        // Check if capability already exists
        let existingControllers = signer.capabilities.storage
            .getControllers(forPath: /storage/vault)

        if existingControllers.length > 0 {
            log("Capability already exists")
            return
        }

        // Issue new capability
        let controller = signer.capabilities.storage
            .issue<&Vault>(/storage/vault)

        // Check if already published
        let existingCap = signer.capabilities
            .borrow<&{FungibleToken.Receiver}>(/public/vault)

        if existingCap == nil {
            // Publish capability
            signer.capabilities.publish(controller.capability, at: /public/vault)
        }
    }
}
```

**Benefits**:
- Prevents duplicate capabilities
- Saves storage space
- Reduces confusion
- Lower costs

### Pattern 11: Capability Revocation

**Problem**: Need to revoke previously issued capabilities.

**Solution**: Store controllers and use them to delete capabilities.

```cadence
// Setup: Store controller during issuance
transaction() {
    prepare(signer: auth(IssueStorageCapabilityController, SaveValue) &Account) {
        // Issue capability
        let controller = signer.capabilities.storage
            .issue<&MyResource>(/storage/myResource)

        // Store controller for later revocation
        let controllers = signer.storage
            .borrow<&[StorageCapabilityController]>(from: /storage/controllers)
            ?? panic("Could not borrow StorageCapabilityController reference from /storage/controllers")

        controllers.append(controller)
    }
}

// Revocation: Delete capability
transaction(capabilityID: UInt64) {
    prepare(signer: auth(Capabilities) &Account) {
        // Delete capability
        signer.capabilities.storage.delete(capabilityID)

        log("Capability revoked")
    }
}

// Revocation: By path
transaction() {
    prepare(signer: auth(Capabilities) &Account) {
        // Get all controllers for path
        let controllers = signer.capabilities.storage
            .getControllers(forPath: /storage/myResource)

        // Revoke all
        for controller in controllers {
            signer.capabilities.storage.delete(controller.capabilityID)
        }
    }
}
```

**Benefits**:
- Unilateral revocation
- No recipient cooperation needed
- Immediate effect
- Security control

## Resource Patterns

### Pattern 12: Resource Wrapper

**Problem**: Need to add functionality to existing resources.

**Solution**: Create wrapper resource that contains original.

```cadence
// Original resource
access(all) resource Token {
    access(all) let id: UInt64
    init(id: UInt64) { self.id = id }
}

// Wrapper adds metadata
access(all) resource EnhancedToken {
    access(self) let token: @Token
    access(all) let metadata: {String: String}
    access(all) var accessCount: UInt64

    access(all) fun getID(): UInt64 {
        self.accessCount = self.accessCount + 1
        return self.token.id
    }

    init(token: @Token, metadata: {String: String}) {
        self.token <- token
        self.metadata = metadata
        self.accessCount = 0
    }

    destroy() {
        destroy self.token
    }
}
```

**Benefits**:
- Adds functionality without modifying original
- Maintains original interface
- Tracks additional state

### Pattern 13: Resource Collection

**Problem**: Need to manage multiple resources of same type.

**Solution**: Create collection resource with dictionary storage.

```cadence
access(all) entitlement Withdraw

access(all) resource Collection {
    access(self) var items: @{UInt64: Item}

    // Deposit item
    access(all) fun deposit(item: @Item) {
        let id = item.id
        let old <- self.items[id] <- item
        destroy old
    }

    // Withdraw item
    access(Withdraw) fun withdraw(id: UInt64): @Item {
        let item <- self.items.remove(key: id)
            ?? panic("Item with ID \(id) not found in collection")
        return <-item
    }

    // Borrow reference
    access(all) fun borrowItem(id: UInt64): &Item? {
        return &self.items[id] as &Item?
    }

    // Get IDs
    access(all) fun getIDs(): [UInt64] {
        return self.items.keys
    }

    init() {
        self.items <- {}
    }

    destroy() {
        destroy self.items
    }
}
```

**Benefits**:
- Standard collection interface
- Safe resource management
- Reference borrowing support
- Enumeration support

## Storage Patterns

### Pattern 14: Storage Path Naming Convention

**Problem**: Inconsistent storage paths cause confusion.

**Solution**: Use descriptive, hierarchical path naming.

```cadence
// ❌ UNCLEAR: Generic names
/storage/vault
/storage/collection
/storage/resource

// ✅ CLEAR: Descriptive names
/storage/flowTokenVault
/storage/exampleNFTCollection
/storage/stakingRewards

// ✅ HIERARCHICAL: Grouped by functionality
/storage/tokens/flow
/storage/tokens/usdc
/storage/nfts/exampleNFT
/storage/nfts/topShot
```

**Benefits**:
- Easy to understand purpose
- Prevents path conflicts
- Organized storage layout

### Pattern 15: Storage Existence Check

**Problem**: Overwriting existing storage without checking.

**Solution**: Always check before saving to storage.

```cadence
// ✅ PATTERN: Check before save
transaction() {
    prepare(signer: auth(BorrowValue, SaveValue) &Account) {
        // Check if path is already used
        let existing = signer.storage
            .borrow<&AnyResource>(from: /storage/myResource)

        if existing != nil {
            panic("Path /storage/myResource is already in use")
        }

        // Safe to save
        let resource <- createResource()
        signer.storage.save(<-resource, to: /storage/myResource)
    }
}
```

**Benefits**:
- Prevents accidental overwrites
- Explicit error handling
- Data safety

## Error Message Patterns

### Pattern 16: Descriptive Panic Messages

**Problem**: Generic error messages make debugging difficult and give users no guidance on how to fix the issue.

**Solution**: Include location context, what failed, why it failed, and how to fix it.

```cadence
// ❌ VAGUE: No context, no guidance
panic("Not found")
panic("Insufficient balance")
panic("Invalid")
panic("Error")

// ✅ DESCRIPTIVE: Context, cause, and actionable information
panic("NFT with ID \(id) not found in collection")
panic("Insufficient balance: available \(self.balance), required \(amount)")
panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault")
panic("Could not borrow FungibleToken Receiver reference from /public/flowTokenReceiver")
```

**Format by operation type**:

| Operation | Format |
|-----------|--------|
| Storage borrow | `"Could not borrow <TypeName> reference from /storage/path"` |
| Capability borrow | `"Could not borrow <TypeName> reference from /public/path"` |
| Storage load | `"<TypeName> not found at /storage/path"` |
| Collection lookup | `"<TypeName> with ID \(id) not found in collection"` |
| Balance check | `"Insufficient balance: available \(self.balance), required \(amount)"` |
| Inbox claim | `"Capability named \"name\" not found in inbox from provider \(provider)"` |

**Type naming**: Use human-readable names like "FungibleToken Vault" or "TopShot Collection" rather than dot notation like "FungibleToken.Vault". This helps non-technical users understand errors.

**Values in messages**: Use string interpolation (`\(value)`) to include actual values. This makes it far easier to diagnose failures without needing additional logging.

```cadence
// ❌ Static message - doesn't tell you what ID was missing
?? panic("NFT not found")

// ✅ Dynamic message - tells you exactly what failed
?? panic("NFT with ID \(id) not found in collection")

// ❌ Doesn't tell you the amounts involved
pre { self.balance >= amount: "Insufficient balance" }

// ✅ Shows both the available and required amounts
pre { self.balance >= amount: "Insufficient balance: available \(self.balance), required \(amount)" }
```

### Pattern 17: Contract Error Message Helpers

**Problem**: Duplicating error message strings across multiple transactions creates inconsistency and typos.

**Solution**: Define reusable error message functions in contracts so all callers share the same wording.

```cadence
access(all) contract MyToken {
    access(all) let vaultStoragePath: StoragePath
    access(all) let receiverPublicPath: PublicPath

    // Error message helpers - callable from transactions and scripts
    access(all) view fun vaultNotFoundError(): String {
        return "Could not borrow MyToken Vault reference from \(self.vaultStoragePath)"
    }

    access(all) view fun receiverNotFoundError(address: Address): String {
        return "Could not borrow MyToken Receiver reference from \(self.receiverPublicPath) for account \(address)"
    }

    access(all) view fun nftNotFoundError(id: UInt64): String {
        return "NFT with ID \(id) not found in collection"
    }

    init() {
        self.vaultStoragePath = /storage/myTokenVault
        self.receiverPublicPath = /public/myTokenReceiver
    }
}

// Usage in transactions - consistent error message, no duplication
transaction(amount: UFix64, recipient: Address) {
    prepare(signer: auth(BorrowValue) &Account) {
        let vault = signer.storage
            .borrow<auth(FungibleToken.Withdraw) &MyToken.Vault>(from: MyToken.vaultStoragePath)
            ?? panic(MyToken.vaultNotFoundError())

        let receiver = getAccount(recipient)
            .capabilities.borrow<&{FungibleToken.Receiver}>(MyToken.receiverPublicPath)
            ?? panic(MyToken.receiverNotFoundError(address: recipient))

        receiver.deposit(from: <-vault.withdraw(amount: amount))
    }
}
```

**Benefits**:
- Single source of truth for error wording
- Consistent messages across all transactions
- Easy to update messaging without touching every transaction
- Error messages are testable

## AI Agent Generation Guidelines

When generating Cadence code, apply these patterns:

1. **Use named constants** for any repeated values
2. **Make data script-accessible** when appropriate
3. **Create report structs** for resources queried by scripts
4. **Use descriptive names** for all identifiers
5. **Use plural names** for collections
6. **Add post-conditions** to verify transaction outcomes
7. **Prefer borrow** over load/save
8. **Check for existing capabilities** before issuing
9. **Store controllers** for revocable capabilities
10. **Check storage paths** before saving
11. **Write descriptive panic messages** with type, path, and interpolated values

## Reference Links

- [Cadence Design Patterns Documentation](https://cadence-lang.org/docs/design-patterns)
- [Cadence Anti-Patterns](https://cadence-lang.org/docs/anti-patterns)
- [Cadence Security Best Practices](https://cadence-lang.org/docs/security-best-practices)
