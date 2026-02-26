# Cadence Contract Rules

> **Purpose**: Define comprehensive guidelines for writing, deploying, and managing Cadence contracts. Contracts are the foundational building blocks of Flow applications.

## Core Philosophy: Contracts as Persistent State Containers

Contracts are **collections of type definitions, data, and functions** stored permanently in account contract storage. They define the rules and capabilities of on-chain applications.

## Contract Structure

### Basic Contract Declaration

```cadence
access(all) contract MyContract {
    // Contract-level declarations

    // Type definitions
    access(all) resource NFT {
        access(all) let id: UInt64
        init(id: UInt64) { self.id = id }
    }

    // State (fields)
    access(self) var totalSupply: UInt64

    // Functions
    access(all) fun getTotalSupply(): UInt64 {
        return self.totalSupply
    }

    // Initializer (runs once on deployment)
    init() {
        self.totalSupply = 0
    }
}
```

### Contract Components

1. **Type Definitions**: Resources, structs, interfaces, enums
2. **State Fields**: Contract-level variables
3. **Functions**: Contract-level functions
4. **Events**: Event declarations
5. **Initializer**: `init()` function (runs once on deployment)

## Contract Initializer

### Purpose and Rules

The contract initializer:
- Runs **exactly once** when the contract is first deployed
- Sets up initial contract state
- Can write to account storage
- Cannot be called again

```cadence
access(all) contract TokenContract {
    access(self) var totalSupply: UFix64
    access(self) var paused: Bool

    init() {
        // Initialize contract state
        self.totalSupply = 0.0
        self.paused = false

        // Create and save admin resource
        let admin <- create Admin()
        self.account.storage.save(<-admin, to: /storage/tokenAdmin)

        // Create and publish public capability
        let publicCap = self.account.capabilities.storage
            .issue<&Vault>(/storage/mainVault)
        self.account.capabilities.publish(publicCap, at: /public/tokenVault)
    }
}
```

### Common Initializer Patterns

**Pattern 1: State Initialization**
```cadence
init() {
    self.totalSupply = 0
    self.counter = 0
    self.paused = false
}
```

**Pattern 2: Admin Resource Creation**
```cadence
init() {
    // Create single admin instance
    let admin <- create Admin()
    self.account.storage.save(<-admin, to: /storage/admin)
}
```

**Pattern 3: Public Capability Setup**
```cadence
init() {
    // Setup public capabilities for common access
    let controller = self.account.capabilities.storage
        .issue<&Resource>(/storage/resource)

    self.account.capabilities.publish(
        controller.capability,
        at: /public/resource
    )
}
```

**Pattern 4: Event Emission**
```cadence
init() {
    self.totalSupply = 0

    // Emit contract initialization event
    emit ContractInitialized()
}
```

## The `account` Field

### Implicit Account Access

Every contract has an implicit `account` field providing full access to the deploying account:

```cadence
access(all) contract MyContract {
    // `self.account` is automatically available

    init() {
        // Access account storage
        let resource <- create MyResource()
        self.account.storage.save(<-resource, to: /storage/myResource)

        // Access account capabilities
        let cap = self.account.capabilities.storage
            .issue<&MyResource>(/storage/myResource)

        // Access account contracts
        log(self.account.contracts.names)
    }

    access(all) fun useAccount() {
        // Can use self.account in any contract function
        let resource = self.account.storage
            .borrow<&MyResource>(from: /storage/myResource)
    }
}
```

### Account Field Capabilities

The `account` field provides access to:
- **Storage**: `self.account.storage`
- **Capabilities**: `self.account.capabilities`
- **Contracts**: `self.account.contracts`
- **Keys**: `self.account.keys`
- **Inbox**: `self.account.inbox`

```cadence
access(all) contract FullExample {
    init() {
        // Storage operations
        let resource <- create MyResource()
        self.account.storage.save(<-resource, to: /storage/myResource)

        // Capability operations
        let cap = self.account.capabilities.storage
            .issue<&MyResource>(/storage/myResource)

        // Check deployed contracts
        log(self.account.contracts.names)

        // Access account address
        log(self.account.address)
    }
}
```

## Contract Interfaces

### Defining Contract Interfaces

Contract interfaces establish behavioral contracts that implementing contracts must follow:

```cadence
// Define interface
access(all) contract interface TokenInterface {
    // Required events
    access(all) event TokensInitialized(initialSupply: UFix64)
    access(all) event TokensWithdrawn(amount: UFix64, from: Address?)

    // Required resources/structs/interfaces
    access(all) resource interface Provider {
        access(all) fun withdraw(amount: UFix64): @Vault
    }

    access(all) resource interface Receiver {
        access(all) fun deposit(from: @Vault)
    }

    // Required fields
    access(all) var totalSupply: UFix64

    // Required functions
    access(all) fun getTotalSupply(): UFix64
}
```

### Implementing Contract Interfaces

```cadence
// Implement interface
access(all) contract MyToken: TokenInterface {
    // Must implement all required events
    access(all) event TokensInitialized(initialSupply: UFix64)
    access(all) event TokensWithdrawn(amount: UFix64, from: Address?)

    // Must implement all required types
    access(all) resource Vault: TokenInterface.Provider, TokenInterface.Receiver {
        access(self) var balance: UFix64

        access(all) fun withdraw(amount: UFix64): @Vault {
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

    // Must implement all required fields
    access(all) var totalSupply: UFix64

    // Must implement all required functions
    access(all) fun getTotalSupply(): UFix64 {
        return self.totalSupply
    }

    init() {
        self.totalSupply = 1000.0
        emit TokensInitialized(initialSupply: 1000.0)
    }
}
```

### Nested Interface Access

Implementing contracts can reference nested interfaces through the interface name:

```cadence
// In contract implementation
access(all) resource MyVault: TokenInterface.Provider, TokenInterface.Receiver {
    // Implementation
}
```

## Contract Deployment and Management

### Deploying Contracts

Contracts are deployed using the `Account.Contracts` API:

```cadence
transaction(contractName: String, code: String) {
    prepare(signer: auth(Contracts) &Account) {
        // Deploy new contract
        signer.contracts.add(
            name: contractName,
            code: code.utf8
        )
    }
}
```

### Updating Contracts

```cadence
transaction(contractName: String, newCode: String) {
    prepare(signer: auth(Contracts) &Account) {
        // Update existing contract
        signer.contracts.update(
            name: contractName,
            code: newCode.utf8
        )
    }
}
```

**Important**: Updates preserve existing data but do not re-run the initializer.

### Removing Contracts

```cadence
transaction(contractName: String) {
    prepare(signer: auth(Contracts) &Account) {
        // Remove contract from account
        signer.contracts.remove(name: contractName)
    }
}
```

### Retrieving Contract Information

```cadence
// Get contract names
let names = account.contracts.names

// Get contract by name
if let contract = account.contracts.get(name: "MyContract") {
    log(contract.code)
    log(contract.name)
}

// Borrow contract (for calling functions)
if let myContract = account.contracts.borrow<&MyContract>(name: "MyContract") {
    myContract.someFunction()
}
```

## Contract Organization Patterns

### Pattern 1: Core + Helpers

```cadence
// Core contract with main functionality
access(all) contract Core {
    access(all) resource MainResource {
        // Core functionality
    }

    init() {
        // Core initialization
    }
}

// Helper contract with utilities
access(all) contract Helpers {
    access(all) fun utilityFunction(): String {
        return "helper"
    }
}
```

### Pattern 2: Contract + Interface

```cadence
// Interface contract (deployed first)
access(all) contract interface IToken {
    access(all) resource interface Provider {
        access(all) fun withdraw(amount: UFix64): @Vault
    }
}

// Implementation contract (deployed later, references interface)
import IToken from 0x01

access(all) contract Token: IToken {
    access(all) resource Vault: IToken.Provider {
        access(all) fun withdraw(amount: UFix64): @Vault {
            // Implementation
        }
    }
}
```

### Pattern 3: Modular Components

```cadence
// Separate contracts for different concerns
access(all) contract TokenLogic {
    // Token transfer logic
}

access(all) contract TokenStorage {
    // Token storage and balances
}

access(all) contract TokenMetadata {
    // Token metadata and views
}
```

## Contract Nesting Rules

### What Can Be Nested

```cadence
access(all) contract MyContract {
    // ✅ Resources
    access(all) resource MyResource { }

    // ✅ Structs
    access(all) struct MyStruct { }

    // ✅ Interfaces
    access(all) resource interface MyInterface { }

    // ✅ Enums
    access(all) enum Status: UInt8 {
        access(all) case active
        access(all) case inactive
    }

    // ✅ Events
    access(all) event MyEvent()

    // ✅ Functions
    access(all) fun myFunction() { }
}
```

### What Cannot Be Nested

```cadence
access(all) contract MyContract {
    // ❌ Cannot nest contracts
    access(all) contract NestedContract { }  // COMPILE ERROR

    // ❌ Cannot nest contract interfaces
    access(all) contract interface NestedInterface { }  // COMPILE ERROR
}
```

## Contract Best Practices

### Practice 1: Single Responsibility

**Each contract should have one clear purpose.**

```cadence
// ✅ GOOD: Focused contract
access(all) contract TokenVault {
    // Only handles vault logic
    access(all) resource Vault {
        // Vault implementation
    }
}

// ❌ LESS IDEAL: Kitchen sink contract
access(all) contract Everything {
    access(all) resource Vault { }
    access(all) resource NFT { }
    access(all) resource Staking { }
    access(all) resource Governance { }
    // Too many responsibilities
}
```

### Practice 2: Use Contract Interfaces

**Define interfaces for composability and upgradeability.**

```cadence
// Interface defines contract
access(all) contract interface FungibleToken {
    access(all) resource interface Provider {
        access(all) fun withdraw(amount: UFix64): @Vault
    }

    access(all) resource interface Receiver {
        access(all) fun deposit(from: @Vault)
    }
}

// Multiple implementations can exist
access(all) contract FlowToken: FungibleToken { }
access(all) contract USDC: FungibleToken { }
```

### Practice 3: Initialize in `init()`

**Set up all contract state in the initializer.**

```cadence
// ✅ CORRECT: Complete initialization
access(all) contract MyContract {
    access(self) var initialized: Bool
    access(self) var counter: UInt64

    init() {
        self.initialized = true
        self.counter = 0

        // Create admin
        let admin <- create Admin()
        self.account.storage.save(<-admin, to: /storage/admin)

        // Emit event
        emit ContractInitialized()
    }
}

// ❌ WRONG: Lazy initialization
access(all) contract MyContract {
    access(self) var initialized: Bool
    access(self) var counter: UInt64

    init() {
        self.initialized = false
        self.counter = 0
        // State not fully set up!
    }

    // Trying to initialize later - but init() only runs once!
    access(all) fun initialize() {
        self.initialized = true  // This is NOT the same as init()
    }
}
```

### Practice 4: Use Events for Observability

**Emit events for significant contract actions.**

```cadence
access(all) contract ObservableContract {
    access(all) event ContractInitialized()
    access(all) event StateUpdated(newValue: UInt64)
    access(all) event AdminActionPerformed(action: String)

    access(self) var state: UInt64

    access(all) fun updateState(newValue: UInt64) {
        self.state = newValue
        emit StateUpdated(newValue: newValue)
    }

    init() {
        self.state = 0
        emit ContractInitialized()
    }
}
```

### Practice 5: Minimal Public Surface

**Expose only what's necessary through public functions.**

```cadence
// ✅ CORRECT: Controlled interface
access(all) contract SecureContract {
    access(self) var internalState: {String: UInt64}

    // Public view function
    access(all) view fun getValue(key: String): UInt64? {
        return self.internalState[key]
    }

    // Entitled modification
    access(Admin) fun setValue(key: String, value: UInt64) {
        self.internalState[key] = value
    }

    init() {
        self.internalState = {}
    }
}
```

## Contract Upgrade Patterns

### Safe Upgrade Guidelines

1. **Preserve field order and types**
2. **Only add new fields at the end**
3. **Don't remove or rename existing fields**
4. **Add new functions freely**
5. **Modify function implementations carefully**

```cadence
// Original contract
access(all) contract MyContract {
    access(all) var fieldA: String
    access(all) var fieldB: UInt64

    access(all) fun functionA() {
        // Implementation
    }

    init() {
        self.fieldA = "initial"
        self.fieldB = 0
    }
}

// ✅ SAFE upgrade
access(all) contract MyContract {
    access(all) var fieldA: String  // Unchanged
    access(all) var fieldB: UInt64  // Unchanged
    access(all) var fieldC: Bool    // New field added at end

    access(all) fun functionA() {
        // Modified implementation OK
    }

    access(all) fun functionB() {
        // New function OK
    }

    init() {
        self.fieldA = "initial"
        self.fieldB = 0
        self.fieldC = false  // Initialize new field
    }
}

// ❌ DANGEROUS upgrade
access(all) contract MyContract {
    access(all) var fieldB: UInt64  // Reordered - BAD
    access(all) var fieldA: UFix64  // Changed type - BAD
    // fieldC removed - BAD

    init() {
        self.fieldA = 0.0
        self.fieldB = 0
    }
}
```

## AI Agent Generation Rules

### Rule 1: Always Include Initializer

**Every contract must have an `init()` function.**

```cadence
// ✅ CORRECT: Has initializer
access(all) contract MyContract {
    access(self) var state: UInt64

    init() {
        self.state = 0
    }
}

// ❌ WRONG: Missing initializer
access(all) contract MyContract {
    access(self) var state: UInt64
    // COMPILE ERROR: No init()
}
```

### Rule 2: Use Contract Interfaces When Standardizing

**Generate contract interfaces for standard functionality.**

```cadence
// Generate interface first
access(all) contract interface StandardInterface {
    access(all) resource interface Resource {
        access(all) fun commonFunction(): String
    }
}

// Then generate implementations
access(all) contract Implementation: StandardInterface {
    access(all) resource MyResource: StandardInterface.Resource {
        access(all) fun commonFunction(): String {
            return "implemented"
        }
    }
}
```

### Rule 3: Initialize All State in `init()`

**Set initial values for all contract fields.**

```cadence
access(all) contract MyContract {
    access(self) var counter: UInt64
    access(self) var name: String
    access(self) var active: Bool

    init() {
        // Initialize ALL fields
        self.counter = 0
        self.name = "MyContract"
        self.active = true
    }
}
```

### Rule 4: Emit Initialization Event

**Signal successful deployment with an event.**

```cadence
access(all) contract MyContract {
    access(all) event ContractInitialized(address: Address)

    init() {
        // Setup...

        emit ContractInitialized(address: self.account.address)
    }
}
```

## Testing Contracts

```cadence
import Test

access(all) fun testContractDeployment() {
    // Deploy contract
    let account = Test.createAccount()
    let code = loadCode("MyContract.cdc")

    let err = account.contracts.add(
        name: "MyContract",
        code: code
    )

    Test.expect(err, Test.equal(nil))

    // Verify contract exists
    let names = account.contracts.names
    Test.expect(names.contains("MyContract"), Test.equal(true))
}

access(all) fun testContractFunctionality() {
    // Borrow contract and test functions
    let contract = account.contracts.borrow<&MyContract>(name: "MyContract")!

    let result = contract.someFunction()
    Test.expect(result, Test.equal(expectedValue))
}
```

## Reference Links

- [Cadence Contracts Documentation](https://cadence-lang.org/docs/language/contracts)
- [Contract Interfaces](https://cadence-lang.org/docs/language/contracts#contract-interfaces)
- [Contract Updatability](https://cadence-lang.org/docs/language/contract-updatability)
- [Account Contracts API](https://cadence-lang.org/docs/language/accounts#accountcontracts)
