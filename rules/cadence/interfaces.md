# Cadence Interface Rules

> **Purpose**: Define comprehensive guidelines for creating and implementing interfaces in Cadence. Interfaces establish behavioral contracts that enable composability, polymorphism, and standardization.

## Core Philosophy: Contracts Over Implementation

Interfaces define **what** must exist without specifying **how** it works. This enables:
- Multiple implementations of the same interface
- Polymorphic functions that work with any implementation
- Standardization across contracts
- Flexible composition

## Three Interface Types

Cadence supports three categories of interfaces:

1. **Struct Interfaces** - Implemented by structs
2. **Resource Interfaces** - Implemented by resources
3. **Contract Interfaces** - Implemented by contracts

## Struct Interfaces

### Defining Struct Interfaces

```cadence
access(all) struct interface NamedEntity {
    // Required field
    access(all) let name: String

    // Required function
    access(all) view fun getName(): String
}
```

### Implementing Struct Interfaces

```cadence
access(all) struct User: NamedEntity {
    access(all) let name: String
    access(all) let id: UInt64

    access(all) view fun getName(): String {
        return self.name
    }

    init(name: String, id: UInt64) {
        self.name = name
        self.id = id
    }
}
```

### Multiple Interface Implementation

```cadence
access(all) struct interface Identifiable {
    access(all) let id: UInt64
    access(all) view fun getID(): UInt64
}

access(all) struct interface Timestamped {
    access(all) let createdAt: UFix64
    access(all) view fun getTimestamp(): UFix64
}

// Implement multiple interfaces
access(all) struct Record: Identifiable, Timestamped {
    access(all) let id: UInt64
    access(all) let createdAt: UFix64

    access(all) view fun getID(): UInt64 {
        return self.id
    }

    access(all) view fun getTimestamp(): UFix64 {
        return self.createdAt
    }

    init(id: UInt64) {
        self.id = id
        self.createdAt = getCurrentBlock().timestamp
    }
}
```

## Resource Interfaces

### Defining Resource Interfaces

```cadence
access(all) resource interface Provider {
    // Required function
    access(all) fun withdraw(amount: UFix64): @Vault

    // Required field
    access(all) var balance: UFix64
}

access(all) resource interface Receiver {
    access(all) fun deposit(from: @Vault)
}
```

### Implementing Resource Interfaces

```cadence
access(all) resource Vault: Provider, Receiver {
    access(all) var balance: UFix64

    access(all) fun withdraw(amount: UFix64): @Vault {
        pre {
            self.balance >= amount: "Insufficient balance: available \(self.balance), required \(amount)"
        }

        self.balance = self.balance - amount
        return <- create Vault(balance: amount)
    }

    access(all) fun deposit(from: @Vault) {
        self.balance = self.balance + from.balance
        destroy from
    }

    init(balance: UFix64) {
        self.balance = balance
    }
}
```

## Contract Interfaces

### Defining Contract Interfaces

```cadence
access(all) contract interface FungibleToken {
    // Required events
    access(all) event TokensInitialized(initialSupply: UFix64)
    access(all) event TokensWithdrawn(amount: UFix64, from: Address?)
    access(all) event TokensDeposited(amount: UFix64, to: Address?)

    // Required resource interfaces
    access(all) resource interface Provider {
        access(all) fun withdraw(amount: UFix64): @Vault
    }

    access(all) resource interface Receiver {
        access(all) fun deposit(from: @Vault)
    }

    access(all) resource interface Balance {
        access(all) var balance: UFix64
    }

    // Required resource type (using nested interfaces)
    access(all) resource Vault: Provider, Receiver, Balance {
        access(all) var balance: UFix64

        access(all) fun withdraw(amount: UFix64): @Vault
        access(all) fun deposit(from: @Vault)
    }

    // Required fields
    access(all) var totalSupply: UFix64

    // Required functions
    access(all) fun createEmptyVault(): @Vault
}
```

### Implementing Contract Interfaces

```cadence
access(all) contract FlowToken: FungibleToken {
    // Must implement all required events
    access(all) event TokensInitialized(initialSupply: UFix64)
    access(all) event TokensWithdrawn(amount: UFix64, from: Address?)
    access(all) event TokensDeposited(amount: UFix64, to: Address?)

    // Must implement required resource
    access(all) resource Vault: FungibleToken.Provider, FungibleToken.Receiver, FungibleToken.Balance {
        access(all) var balance: UFix64

        access(all) fun withdraw(amount: UFix64): @Vault {
            self.balance = self.balance - amount
            emit TokensWithdrawn(amount: amount, from: self.owner?.address)
            return <- create Vault(balance: amount)
        }

        access(all) fun deposit(from: @Vault) {
            let vault <- from as! @FlowToken.Vault
            self.balance = self.balance + vault.balance
            emit TokensDeposited(amount: vault.balance, to: self.owner?.address)
            destroy vault
        }

        init(balance: UFix64) {
            self.balance = balance
        }
    }

    // Must implement required fields
    access(all) var totalSupply: UFix64

    // Must implement required functions
    access(all) fun createEmptyVault(): @Vault {
        return <- create Vault(balance: 0.0)
    }

    init() {
        self.totalSupply = 1000.0
        emit TokensInitialized(initialSupply: self.totalSupply)
    }
}
```

## Interface Requirements

### Field Requirements

Interfaces can specify required fields:

```cadence
access(all) resource interface NFTPublic {
    // Required field - must be public
    access(all) let id: UInt64

    // Required field with specific type
    access(all) let metadata: {String: String}
}

// Implementation must match exactly
access(all) resource NFT: NFTPublic {
    access(all) let id: UInt64
    access(all) let metadata: {String: String}

    init(id: UInt64, metadata: {String: String}) {
        self.id = id
        self.metadata = metadata
    }
}
```

**Rules**:
- Interface fields must be `access(all)`
- Implementation fields must match:
  - Name
  - Type
  - Variable vs constant (`var` vs `let`)
- Implementation can be more permissive (but not less)

### Function Requirements

Interfaces can specify required functions:

```cadence
access(all) resource interface Collection {
    // Required function signature
    access(all) view fun getIDs(): [UInt64]

    // Required function with parameters
    access(all) view fun borrowNFT(id: UInt64): &NFT?

    // Required function with pre-condition
    access(all) fun deposit(token: @NFT) {
        pre {
            token.id != nil: "Token must have ID"
        }
    }
}
```

**Rules**:
- Interface functions must be `access(all)` (minimum)
- Implementation must match:
  - Function name
  - Parameter names and types
  - Return type
- Implementation can add additional logic
- Pre/post conditions in interface are inherited

### Pre and Post Conditions in Interfaces

Interfaces can define conditions that implementations must satisfy:

```cadence
access(all) resource interface Vault {
    access(all) var balance: UFix64

    access(all) fun withdraw(amount: UFix64): @Vault {
        pre {
            self.balance >= amount: "Insufficient balance: available \(self.balance), required \(amount)"
        }
        post {
            self.balance == before(self.balance) - amount:
                "Balance not updated correctly"
        }
    }
}

// Implementation inherits these conditions
access(all) resource MyVault: Vault {
    access(all) var balance: UFix64

    // Must satisfy interface pre/post conditions
    access(all) fun withdraw(amount: UFix64): @Vault {
        // Interface pre-condition checked first
        // Additional implementation logic
        self.balance = self.balance - amount
        return <- create MyVault(balance: amount)
        // Interface post-condition checked last
    }

    init(balance: UFix64) {
        self.balance = balance
    }
}
```

## Default Functions

Interfaces can provide default implementations:

```cadence
access(all) resource interface Counter {
    access(all) var count: UInt64

    // Default implementation
    access(all) fun increment() {
        self.count = self.count + 1
    }

    // Default implementation
    access(all) view fun getCount(): UInt64 {
        return self.count
    }
}

// Implementation inherits default functions
access(all) resource MyCounter: Counter {
    access(all) var count: UInt64

    // Can use inherited increment() without implementing
    // Can override if needed

    init() {
        self.count = 0
    }
}

// Usage
let counter <- create MyCounter()
counter.increment()  // Uses default implementation
log(counter.getCount())  // Uses default implementation
```

### Overriding Default Functions

```cadence
access(all) resource MyCustomCounter: Counter {
    access(all) var count: UInt64

    // Override default implementation
    access(all) fun increment() {
        self.count = self.count + 2  // Custom behavior
    }

    // Use default getCount()

    init() {
        self.count = 0
    }
}
```

## Interface Inheritance

Interfaces can inherit from other interfaces of the same kind:

```cadence
// Base interface
access(all) resource interface Base {
    access(all) view fun baseFunction(): String
}

// Derived interface
access(all) resource interface Extended: Base {
    access(all) view fun extendedFunction(): String
}

// Implementation must satisfy both
access(all) resource MyResource: Extended {
    access(all) view fun baseFunction(): String {
        return "base"
    }

    access(all) view fun extendedFunction(): String {
        return "extended"
    }
}
```

### Multiple Inheritance

```cadence
access(all) resource interface A {
    access(all) view fun methodA(): String
}

access(all) resource interface B {
    access(all) view fun methodB(): String
}

access(all) resource interface C: A, B {
    access(all) view fun methodC(): String
}

// Must implement all methods
access(all) resource MyResource: C {
    access(all) view fun methodA(): String { return "A" }
    access(all) view fun methodB(): String { return "B" }
    access(all) view fun methodC(): String { return "C" }
}
```

## Intersection Types

### Using Intersection Types

Intersection types represent values that implement specific interfaces:

```cadence
// Interface
access(all) resource interface Provider {
    access(all) fun withdraw(amount: UFix64): @Vault
}

// Function using intersection type
access(all) fun withdrawFromProvider(
    provider: &{Provider},  // Any resource implementing Provider
    amount: UFix64
): @Vault {
    return <- provider.withdraw(amount: amount)
}

// Works with any implementation
let vault1: @Vault <- // ...
let provider1 = &vault1 as &{Provider}
withdrawFromProvider(provider: provider1, amount: 10.0)

let vault2: @CustomVault <- // ...
let provider2 = &vault2 as &{Provider}
withdrawFromProvider(provider: provider2, amount: 10.0)
```

### Multiple Interfaces in Intersection

```cadence
// Multiple interfaces
access(all) resource interface Provider {
    access(all) fun withdraw(amount: UFix64): @Vault
}

access(all) resource interface Balance {
    access(all) var balance: UFix64
}

// Require both interfaces
access(all) fun getBalanceAndWithdraw(
    vault: &{Provider, Balance},  // Must implement both
    amount: UFix64
): UFix64 {
    let balance = vault.balance
    let withdrawn <- vault.withdraw(amount: amount)
    destroy withdrawn
    return balance
}
```

## Nominal Typing

**Cadence uses nominal typing for interfaces**. A type only implements an interface if it explicitly declares conformance:

```cadence
// Interface
access(all) resource interface Named {
    access(all) let name: String
    access(all) view fun getName(): String
}

// ❌ DOES NOT implement Named (missing declaration)
access(all) resource Person {
    access(all) let name: String

    access(all) view fun getName(): String {
        return self.name
    }

    init(name: String) {
        self.name = name
    }
}

// ✅ Implements Named (explicit declaration)
access(all) resource Person: Named {
    access(all) let name: String

    access(all) view fun getName(): String {
        return self.name
    }

    init(name: String) {
        self.name = name
    }
}
```

## Nested Interfaces

Contracts and resources can contain nested interfaces:

```cadence
access(all) contract MyContract {
    // Nested resource interface
    access(all) resource interface VaultPublic {
        access(all) view fun getBalance(): UFix64
    }

    // Resource using nested interface
    access(all) resource Vault: VaultPublic {
        access(self) var balance: UFix64

        access(all) view fun getBalance(): UFix64 {
            return self.balance
        }

        init(balance: UFix64) {
            self.balance = balance
        }
    }
}

// Reference nested interface
let vaultRef: &{MyContract.VaultPublic} = // ...
```

## Interface Best Practices

### Practice 1: Design for Composability

**Create focused interfaces that can be combined.**

```cadence
// ✅ GOOD: Small, focused interfaces
access(all) resource interface Identifiable {
    access(all) let id: UInt64
}

access(all) resource interface Ownable {
    access(all) let owner: Address
}

access(all) resource interface Transferrable {
    access(all) fun transfer(to: Address)
}

// Combine as needed
access(all) resource NFT: Identifiable, Ownable, Transferrable {
    // Implementation
}

// ❌ LESS IDEAL: Monolithic interface
access(all) resource interface EverythingInterface {
    access(all) let id: UInt64
    access(all) let owner: Address
    access(all) let metadata: {String: String}
    access(all) let rarity: String
    access(all) fun transfer(to: Address)
    access(all) fun updateMetadata(key: String, value: String)
    access(all) view fun calculateValue(): UFix64
    // Too many responsibilities
}
```

### Practice 2: Use Contract Interfaces for Standards

**Define contract interfaces for protocol standards.**

```cadence
// Standard interface
access(all) contract interface FungibleTokenStandard {
    access(all) resource interface Provider {
        access(all) fun withdraw(amount: UFix64): @Vault
    }

    access(all) resource interface Receiver {
        access(all) fun deposit(from: @Vault)
    }

    access(all) resource Vault: Provider, Receiver
}

// Multiple implementations follow standard
access(all) contract FlowToken: FungibleTokenStandard { }
access(all) contract USDC: FungibleTokenStandard { }
access(all) contract CustomToken: FungibleTokenStandard { }
```

### Practice 3: Include Pre/Post Conditions

**Define invariants in interface conditions.**

```cadence
access(all) resource interface SafeVault {
    access(all) var balance: UFix64

    access(all) fun withdraw(amount: UFix64): @Vault {
        pre {
            amount > 0.0: "Amount must be positive"
            self.balance >= amount: "Insufficient balance: available \(self.balance), required \(amount)"
        }
        post {
            self.balance == before(self.balance) - amount:
                "Balance not updated correctly"
        }
    }
}
```

### Practice 4: Provide Default Implementations

**Use default functions for common behavior.**

```cadence
access(all) resource interface Collection {
    access(all) var items: @{UInt64: Item}

    // Default implementation
    access(all) view fun getIDs(): [UInt64] {
        return self.items.keys
    }

    // Default implementation
    access(all) view fun getLength(): Int {
        return self.items.length
    }

    // Required implementation (no default)
    access(all) view fun borrowItem(id: UInt64): &Item?
}
```

### Practice 5: Use Intersection Types for Flexibility

**Accept intersection types in function parameters.**

```cadence
// ✅ GOOD: Flexible function
access(all) fun processProvider(provider: &{Provider}) {
    // Works with any Provider implementation
}

// ❌ LESS FLEXIBLE: Concrete type
access(all) fun processVault(vault: &Vault) {
    // Only works with exact Vault type
}
```

## AI Agent Generation Rules

### Rule 1: Generate Interfaces for Standards

**When creating standard functionality, generate interface first.**

```cadence
// Generate interface
access(all) contract interface TokenStandard {
    access(all) resource interface Vault {
        access(all) var balance: UFix64
        access(all) fun withdraw(amount: UFix64): @Vault
        access(all) fun deposit(from: @Vault)
    }
}

// Then generate implementation
access(all) contract MyToken: TokenStandard {
    access(all) resource Vault: TokenStandard.Vault {
        // Implementation
    }
}
```

### Rule 2: Use Intersection Types for Parameters

**Accept intersection types for polymorphic functions.**

```cadence
// Generate flexible function signatures
access(all) fun transfer(
    from: &{Provider, Balance},
    to: &{Receiver},
    amount: UFix64
) {
    // Works with any implementations
}
```

### Rule 3: Include Required Members

**Ensure implementations include all required interface members.**

```cadence
// Interface defines requirements
access(all) resource interface NFT {
    access(all) let id: UInt64
    access(all) view fun getID(): UInt64
}

// Generated implementation must include all
access(all) resource MyNFT: NFT {
    access(all) let id: UInt64  // Required

    access(all) view fun getID(): UInt64 {  // Required
        return self.id
    }

    init(id: UInt64) {
        self.id = id
    }
}
```

### Rule 4: Add Pre/Post Conditions

**Generate interfaces with safety conditions.**

```cadence
access(all) resource interface Vault {
    access(all) var balance: UFix64

    access(all) fun withdraw(amount: UFix64): @Vault {
        pre {
            amount > 0.0: "Amount must be positive"
            self.balance >= amount: "Insufficient balance: available \(self.balance), required \(amount)"
        }
        post {
            self.balance == before(self.balance) - amount:
                "Balance mismatch"
        }
    }
}
```

## Testing Interfaces

```cadence
import Test

access(all) fun testInterfaceConformance() {
    let vault <- createVault()

    // Test that vault implements interface
    let provider = &vault as &{Provider}
    Test.expect(provider != nil, Test.equal(true))

    // Test interface methods work
    let withdrawn <- provider.withdraw(amount: 10.0)
    Test.expect(withdrawn.balance, Test.equal(10.0))

    destroy withdrawn
    destroy vault
}

access(all) fun testIntersectionType() {
    let vault <- createVault()

    // Test multiple interfaces
    let ref = &vault as &{Provider, Receiver, Balance}
    Test.expect(ref != nil, Test.equal(true))

    destroy vault
}
```

## Reference Links

- [Cadence Interfaces Documentation](https://cadence-lang.org/docs/language/interfaces)
- [Contract Interfaces](https://cadence-lang.org/docs/language/contracts#contract-interfaces)
- [Intersection Types](https://cadence-lang.org/docs/language/interfaces#intersection-types)
- [Interface Inheritance](https://cadence-lang.org/docs/language/interfaces#interface-inheritance)
