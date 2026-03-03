# Cadence Account Rules

> **Purpose**: Define comprehensive guidelines for working with accounts in Cadence. Accounts are the fundamental storage and identity mechanism on Flow blockchain.

## Core Philosophy: Accounts as Secure Storage Containers

Accounts in Flow are **multi-faceted entities** that manage:
- **Storage**: Resources and data
- **Capabilities**: Access delegation
- **Keys**: Authentication
- **Contracts**: Deployed code
- **Inbox**: Cross-account communication

## Account Structure

### The `Account` Type

Every account is accessed through an `Account` reference:

```cadence
// In transaction
transaction() {
    prepare(signer: &Account) {
        // signer is an Account reference
        log(signer.address)
    }
}

// Getting any account
let account = getAccount(0x01)

// Getting an authorized reference to an account (Only usable in scripts)
let authAccount = getAuthAccount<auth(Storage) &Account>(0x01)
```

### Account Components

```cadence
account.address        // Account address
account.storage        // Storage management
account.capabilities   // Capability management
account.keys           // Key management
account.contracts      // Contract management
account.inbox          // Inbox for capability bootstrapping
```

## Account Storage

### Storage Paths

Storage paths define where data is stored:

```cadence
// Storage path types
/storage/...  // Private storage
/public/...   // Public capabilities
/private/...  // Private capabilities (deprecated)
```

### Storage Operations

**Saving Resources**:
```cadence
transaction() {
    prepare(signer: auth(SaveValue) &Account) {
        let resource <- createResource()
        signer.storage.save(<-resource, to: /storage/myResource)
    }
}
```

**Loading Resources**:
```cadence
transaction() {
    prepare(signer: auth(LoadValue) &Account) {
        let resource <- signer.storage.load<@MyResource>(from: /storage/myResource)
            ?? panic("Resource of type @MyResource not found at path /storage/myResource")

        // Use resource
        destroy resource
    }
}
```

**Borrowing References**:
```cadence
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        let resourceRef = signer.storage
            .borrow<&MyResource>(from: /storage/myResource)
            ?? panic("Could not borrow MyResource reference from /storage/myResource")

        // Use reference (resource stays in storage)
        resourceRef.someFunction()
    }
}
```

**Copying Values** (structs only):
```cadence
transaction() {
    prepare(signer: auth(CopyValue) &Account) {
        let data = signer.storage.copy<MyStruct>(from: /storage/myData)
            ?? panic("Data of type MyStruct not found at path /storage/myData")

        // Original remains in storage
        log(data)
    }
}
```

### Storage Inspection

```cadence
// Check if path is used
let exists = account.storage.check<@MyResource>(/storage/myResource)

// Get all storage paths
let paths = account.storage.storagePaths

// Iterate storage
for path in account.storage.storagePaths {
    log(path.toString())
}
```

## Account Capabilities

### Capability Paths

```cadence
// Public capabilities
account.capabilities.get<&T>(/public/capability)
account.capabilities.publish(cap, at: /public/capability)

// Storage capabilities (issued to specific storage paths)
account.capabilities.storage.issue<&T>(/storage/resource)
account.capabilities.storage.getControllers(forPath: /storage/resource)

// Account capabilities (issued for account access)
account.capabilities.account.issue<&Account>()
```

### Issuing Storage Capabilities

```cadence
transaction() {
    prepare(signer: auth(IssueStorageCapabilityController) &Account) {
        // Issue capability for storage path
        let controller = signer.capabilities.storage
            .issue<&MyResource>(/storage/myResource)

        controller.setTag("My capability for external access")

        // Get the capability from controller
        let cap = controller.capability
    }
}
```

### Publishing Capabilities

```cadence
transaction() {
    prepare(signer: auth(IssueStorageCapabilityController, PublishCapability) &Account) {
        // Issue capability
        let controller = signer.capabilities.storage
            .issue<&Resource>(/storage/resource)

        // Publish for public access
        signer.capabilities.publish(controller.capability, at: /public/resource)
    }
}
```

### Getting Published Capabilities

```cadence
// Get capability from any account
let cap = getAccount(address)
    .capabilities.get<&{PublicInterface}>(/public/resource)

if cap.check() {
    if let ref = cap.borrow() {
        ref.publicFunction()
    }
}
```

### Unpublishing Capabilities

```cadence
transaction() {
    prepare(signer: auth(UnpublishCapability) &Account) {
        signer.capabilities.unpublish(/public/resource)
    }
}
```

### Capability Controllers

**Getting Controllers**:
```cadence
let controllers = account.capabilities.storage
    .getControllers(forPath: /storage/resource)

for controller in controllers {
    log(controller.capabilityID)
    log(controller.borrowType)
    log(controller.tag)
}
```

**Deleting Capabilities**:
```cadence
transaction() {
    prepare(signer: auth(Capabilities) &Account) {
        let controllers = signer.capabilities.storage
            .getControllers(forPath: /storage/resource)

        for controller in controllers {
            signer.capabilities.storage.delete(controller.capabilityID)
        }
    }
}
```

**Retargeting Capabilities** (storage capabilities only):
```cadence
transaction() {
    prepare(signer: auth(Capabilities, SaveValue) &Account) {
        // Move resource to new path
        let resource <- signer.storage.load<@MyResource>(from: /storage/oldPath)!
        signer.storage.save(<-resource, to: /storage/newPath)

        // Retarget capability
        let controllers = signer.capabilities.storage
            .getControllers(forPath: /storage/oldPath)

        for controller in controllers {
            controller.retarget(/storage/newPath)
        }
    }
}
```

## Account Keys

### Key Management Operations

**Adding Keys**:
```cadence
transaction(publicKey: String, hashAlgorithm: HashAlgorithm, weight: UFix64) {
    prepare(signer: auth(AddKey) &Account) {
        let key = PublicKey(
            publicKey: publicKey.decodeHex(),
            signatureAlgorithm: SignatureAlgorithm.ECDSA_P256
        )

        signer.keys.add(
            publicKey: key,
            hashAlgorithm: hashAlgorithm,
            weight: weight
        )
    }
}
```

**Revoking Keys**:
```cadence
transaction(keyIndex: Int) {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: keyIndex)
            ?? panic("Key not found at index \(keyIndex)")
    }
}
```

**Getting Keys**:
```cadence
// Get specific key
let key = account.keys.get(keyIndex: 0)
    ?? panic("Key not found at index 0")

log(key.publicKey)
log(key.hashAlgorithm)
log(key.weight)
log(key.isRevoked)

// Count keys
let count = account.keys.count
```

## Account Contracts

### Deploying Contracts

```cadence
transaction(name: String, code: String) {
    prepare(signer: auth(Contracts) &Account) {
        signer.contracts.add(
            name: name,
            code: code.utf8
        )
    }
}
```

### Updating Contracts

```cadence
transaction(name: String, newCode: String) {
    prepare(signer: auth(Contracts) &Account) {
        signer.contracts.update(
            name: name,
            code: newCode.utf8
        )
    }
}
```

### Removing Contracts

```cadence
transaction(name: String) {
    prepare(signer: auth(Contracts) &Account) {
        signer.contracts.remove(name: name)
            ?? panic("Contract with name \(name) not found in account \(signer.address)")
    }
}
```

### Getting Contract Information

```cadence
// List contract names
let names = account.contracts.names

// Get contract details
if let contract = account.contracts.get(name: "MyContract") {
    log(contract.name)
    log(String.fromUTF8(contract.code) ?? "")
}

// Borrow contract reference
if let contractRef = account.contracts.borrow<&MyContract>(name: "MyContract") {
    contractRef.someFunction()
}
```

## Account Inbox

### Publishing to Inbox

```cadence
transaction(recipientAddress: Address) {
    prepare(signer: auth(IssueStorageCapabilityController, Inbox) &Account) {
        // Create capability
        let controller = signer.capabilities.storage
            .issue<&MyResource>(/storage/myResource)

        // Publish to recipient's inbox
        signer.inbox.publish(
            controller.capability,
            name: "myResource",
            recipient: recipientAddress
        )
    }
}
```

### Claiming from Inbox

```cadence
transaction(providerAddress: Address, name: String) {
    prepare(signer: auth(Inbox) &Account) {
        // Claim capability from inbox
        let cap = signer.inbox.claim<&MyResource>(
            name: name,
            provider: providerAddress
        ) ?? panic("Capability of type &MyResource not found in inbox")

        // Use capability
        if let ref = cap.borrow() {
            ref.someFunction()
        }
    }
}
```

### Unpublishing from Inbox

```cadence
transaction(recipientAddress: Address, name: String) {
    prepare(signer: auth(Inbox) &Account) {
        signer.inbox.unpublish<&MyResource>(
            name: name,
            recipient: recipientAddress
        )
    }
}
```

## Account Entitlements

### Coarse-Grained Entitlements

Grant access to entire API groups:

```cadence
auth(Storage) &Account              // All storage operations
auth(Contracts) &Account            // All contract operations
auth(Keys) &Account                 // All key operations
auth(Inbox) &Account                // All inbox operations
auth(Capabilities) &Account         // All capability operations
```

### Fine-Grained Entitlements

Grant access to specific operations:

```cadence
// Storage entitlements
auth(SaveValue) &Account
auth(LoadValue) &Account
auth(BorrowValue) &Account
auth(CopyValue) &Account

// Contract entitlements
auth(Contracts) &Account

// Key entitlements
auth(AddKey) &Account
auth(RevokeKey) &Account

// Capability entitlements
auth(IssueStorageCapabilityController) &Account
auth(IssueAccountCapabilityController) &Account
auth(PublishCapability) &Account
auth(UnpublishCapability) &Account

// Inbox entitlements
auth(Inbox) &Account
```

### Combining Entitlements

```cadence
// Multiple specific entitlements
prepare(signer: auth(BorrowValue, SaveValue) &Account) {
    // Can borrow and save
}

// Coarse + fine-grained
prepare(signer: auth(Storage, Contracts) &Account) {
    // All storage operations + contract operations
}
```

## Creating New Accounts

```cadence
transaction() {
    prepare(payer: auth(BorrowValue, Contracts, Storage) &Account) {
        // Create new account (payer pays for creation)
        let newAccount = Account(payer: payer)

        log(newAccount.address)

        // New account has full access initially
        // Set up initial state
        let resource <- createInitialResource()
        newAccount.storage.save(<-resource, to: /storage/initial)
    }
}
```

## Account Best Practices

### Practice 1: Minimal Entitlements

**Request only necessary entitlements in transactions.**

```cadence
// ✅ CORRECT: Minimal entitlements
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Only needs BorrowValue
        let ref = signer.storage.borrow<&MyResource>(from: /storage/resource)
    }
}

// ❌ WRONG: Over-privileged
transaction() {
    prepare(signer: auth(Storage, Contracts, Keys) &Account) {
        // Only uses BorrowValue
        let ref = signer.storage.borrow<&MyResource>(from: /storage/resource)
    }
}
```

### Practice 2: Use Capabilities for Delegation

**Don't pass account references; use capabilities.**

```cadence
// ❌ WRONG: Passing account reference
access(all) fun dangerousFunction(account: auth(Storage) &Account) {
    // Full storage access - dangerous!
}

// ✅ CORRECT: Use capability
access(all) fun safeFunction(cap: Capability<&MyResource>) {
    // Limited, revocable access
    if let ref = cap.borrow() {
        ref.someFunction()
    }
}
```

### Practice 3: Check Storage Before Saving

**Verify path is empty before saving.**

```cadence
// ✅ CORRECT: Check before saving
transaction() {
    prepare(signer: auth(SaveValue, BorrowValue) &Account) {
        // Check if path is already used
        if signer.storage.borrow<&AnyResource>(from: /storage/resource) != nil {
            panic("Path /storage/resource is already in use")
        }

        let resource <- createResource()
        signer.storage.save(<-resource, to: /storage/resource)
    }
}
```

### Practice 4: Clean Up Storage

**Remove unused resources to free storage.**

```cadence
transaction() {
    prepare(signer: auth(LoadValue) &Account) {
        // Load and destroy unused resource
        if let old <- signer.storage.load<@OldResource>(from: /storage/old) {
            destroy old
        }
    }
}
```

### Practice 5: Tag Capabilities

**Use descriptive tags for capability management.**

```cadence
transaction() {
    prepare(signer: auth(IssueStorageCapabilityController) &Account) {
        let controller = signer.capabilities.storage
            .issue<&MyResource>(/storage/resource)

        // Add descriptive tag
        controller.setTag("Public read-only access to MyResource")
    }
}
```

## AI Agent Generation Rules

### Rule 1: Use Minimal Entitlements

**Generate transactions with least-privilege entitlements.**

```cadence
// Generate appropriate entitlements
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // BorrowValue only
    }
}

transaction() {
    prepare(signer: auth(BorrowValue, SaveValue) &Account) {
        // Both needed
    }
}
```

### Rule 2: Always Check Capabilities

**Generate defensive capability borrowing.**

```cadence
// Always check before borrowing
let cap = getAccount(address)
    .capabilities.get<&{MyInterface}>(/public/resource)

if let ref = cap.borrow() {
    ref.someFunction()
} else {
    panic("Could not borrow MyInterface reference from /public/resource")
}
```

### Rule 3: Verify Storage Paths

**Check paths before saving.**

```cadence
transaction() {
    prepare(signer: auth(BorrowValue, SaveValue) &Account) {
        if signer.storage.borrow<&AnyResource>(from: /storage/path) != nil {
            panic("Path /storage/path already occupied")
        }

        let resource <- createResource()
        signer.storage.save(<-resource, to: /storage/path)
    }
}
```

### Rule 4: Use Inbox for Cross-Account Setup

**Generate inbox usage for capability bootstrapping.**

```cadence
// Publisher transaction
transaction(recipient: Address) {
    prepare(signer: auth(IssueStorageCapabilityController, Inbox) &Account) {
        let controller = signer.capabilities.storage
            .issue<&MyResource>(/storage/resource)

        signer.inbox.publish(
            controller.capability,
            name: "myResource",
            recipient: recipient
        )
    }
}

// Claimer transaction
transaction(provider: Address) {
    prepare(signer: auth(Inbox, SaveValue) &Account) {
        let cap = signer.inbox.claim<&MyResource>(
            name: "myResource",
            provider: provider
        ) ?? panic("Capability named \"myResource\" not found in inbox from provider \(provider)")

        // Save capability
        signer.storage.save(cap, to: /storage/receivedCap)
    }
}
```

## Common Entitlement Combinations

```cadence
// Read-only access
auth(BorrowValue) &Account

// Read and write
auth(BorrowValue, SaveValue) &Account

// Capability setup
auth(BorrowValue, IssueStorageCapabilityController, PublishCapability) &Account

// Contract deployment
auth(Contracts) &Account

// Account setup (comprehensive)
auth(BorrowValue, SaveValue, IssueStorageCapabilityController, PublishCapability, Contracts) &Account

// Key management
auth(Keys) &Account
auth(AddKey, RevokeKey) &Account
```

## Testing Accounts

```cadence
import Test

access(all) fun testAccountStorage() {
    let account = Test.createAccount()

    // Test saving
    let resource <- createResource()
    account.storage.save(<-resource, to: /storage/test)

    // Test borrowing
    let ref = account.storage.borrow<&MyResource>(from: /storage/test)
    Test.expect(ref != nil, Test.equal(true))

    // Test loading
    let loaded <- account.storage.load<@MyResource>(from: /storage/test)
    Test.expect(loaded != nil, Test.equal(true))

    destroy loaded
}

access(all) fun testCapabilities() {
    let account = Test.createAccount()

    // Setup resource
    let resource <- createResource()
    account.storage.save(<-resource, to: /storage/test)

    // Issue and publish capability
    let controller = account.capabilities.storage
        .issue<&MyResource>(/storage/test)

    account.capabilities.publish(controller.capability, at: /public/test)

    // Test getting capability
    let cap = account.capabilities.get<&MyResource>(/public/test)
    Test.expect(cap.check(), Test.equal(true))

    // Test borrowing
    if let ref = cap.borrow() {
        Test.assert(true, message: "Successfully borrowed")
    } else {
        Test.assert(false, message: "Failed to borrow")
    }
}
```

## Reference Links

- [Cadence Accounts Documentation](https://cadence-lang.org/docs/language/accounts)
- [Account Entitlements](https://cadence-lang.org/docs/language/accounts#account-entitlements)
- [Storage API](https://cadence-lang.org/docs/language/accounts#account-storage)
- [Capabilities API](https://cadence-lang.org/docs/language/accounts#account-capabilities)
- [Contracts API](https://cadence-lang.org/docs/language/accounts#account-contracts)
