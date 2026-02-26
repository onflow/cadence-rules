# Cadence Transaction Rules

> **Purpose**: Define comprehensive guidelines for writing secure, correct, and maintainable Cadence transactions. Transactions are signed objects that modify blockchain state and must follow strict structural and security patterns.

## Core Philosophy: Explicit Phases for Auditability

Transactions use **explicit execution phases** to separate concerns and make security implications clear to signers. Each phase has specific capabilities and restrictions.

## Transaction Structure

A Cadence transaction consists of:
- **Import statements** (optional)
- **Parameters** (optional)
- **Field declarations** (optional)
- **Four execution phases**: `prepare`, `pre`, `execute`, `post`

```cadence
import "FungibleToken"
import "FlowToken"

transaction(amount: UFix64, recipient: Address) {
    // Field declarations
    let senderVault: &{FungibleToken.Provider}
    let recipientReceiver: &{FungibleToken.Receiver}

    prepare(signer: auth(BorrowValue) &Account) {
        // Access signer account
        self.senderVault = signer.storage
            .borrow<&{FungibleToken.Provider}>(from: /storage/flowTokenVault)
            ?? panic("Could not borrow FungibleToken Provider reference from /storage/flowTokenVault")

        // Access recipient account (public)
        self.recipientReceiver = getAccount(recipient)
            .capabilities.borrow<&{FungibleToken.Receiver}>(/public/flowTokenReceiver)
            ?? panic("Could not borrow FungibleToken Receiver reference from /public/flowTokenReceiver")
    }

    pre {
        amount > 0.0: "Amount must be positive"
    }

    execute {
        let vault <- self.senderVault.withdraw(amount: amount)
        self.recipientReceiver.deposit(from: <-vault)
    }

    post {
        self.senderVault.balance >= 0.0: "Sender balance cannot be negative"
    }
}
```

## The Four Phases

### Phase 1: `prepare`

**Purpose**: Access signing accounts and set up transaction state.

**Key Rules**:
- **ONLY place where signing accounts are accessible**
- Receives authorized account references for each signer
- Use for operations requiring account write access:
  - Borrowing from storage
  - Creating capabilities
  - Issuing controllers
  - Saving resources
- Minimize logic - only account-access operations

```cadence
prepare(signer: auth(BorrowValue, SaveValue, IssueStorageCapabilityController) &Account) {
    // ✅ CORRECT: Borrowing from storage
    self.vault = signer.storage
        .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault")

    // ✅ CORRECT: Issuing capability
    self.cap = signer.capabilities.storage
        .issue<&MyResource>(/storage/myResource)

    // ✅ CORRECT: Saving resource
    let resource <- create MyResource()
    signer.storage.save(<-resource, to: /storage/myResource)

    // ❌ WRONG: Business logic in prepare
    for i in 0...100 {
        // Complex calculations don't need account access
        // Move to execute phase
    }
}
```

**Multiple Signers**:

```cadence
prepare(
    signer1: auth(BorrowValue) &Account,
    signer2: auth(BorrowValue) &Account
) {
    // Access both accounts
    self.vault1 = signer1.storage
        .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault for signer1")

    self.vault2 = signer2.storage
        .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault for signer2")
}
```

### Phase 2: `pre` (Pre-conditions)

**Purpose**: Validate inputs and preconditions before execution.

**Key Rules**:
- Executes AFTER `prepare`, BEFORE `execute`
- Contains boolean expressions with optional error messages
- **View context only** - no state modifications
- If any condition fails, transaction reverts
- All previous phases (prepare) are rolled back

```cadence
pre {
    amount > 0.0: "Amount must be positive"
    amount <= 1000.0: "Amount exceeds maximum"
    recipient != signer.address: "Cannot send to yourself"
}
```

**Multiple Conditions**:

```cadence
pre {
    amount > 0.0: "Amount must be positive"
    self.senderVault.balance >= amount: "Insufficient balance"
    self.recipientReceiver != nil: "Invalid recipient"
}
```

### Phase 3: `execute`

**Purpose**: Main transaction logic and state modifications.

**Key Rules**:
- Executes AFTER `pre`, BEFORE `post`
- Contains primary business logic
- **Cannot access signing accounts directly**
- Can access ANY account's public information via `getAccount()`
- Can call functions, modify state, emit events
- No signer references available here

```cadence
execute {
    // ✅ CORRECT: Business logic
    let vault <- self.senderVault.withdraw(amount: amount)
    self.recipientReceiver.deposit(from: <-vault)

    // ✅ CORRECT: Emit events
    emit TokensTransferred(amount: amount, from: signer.address, to: recipient)

    // ✅ CORRECT: Access public account information
    let recipientBalance = getAccount(recipient)
        .capabilities.borrow<&{FungibleToken.Balance}>(/public/flowTokenBalance)
        ?.balance

    // ❌ WRONG: Cannot access signer account
    signer.storage.save(...) // COMPILE ERROR: signer not available
}
```

### Phase 4: `post` (Post-conditions)

**Purpose**: Verify transaction produced expected results.

**Key Rules**:
- Executes AFTER `execute`
- Contains boolean expressions with error messages
- **View context only** - no state modifications
- Can use `before()` to reference pre-execution values
- If any condition fails, entire transaction reverts
- Useful for documenting invariants

```cadence
post {
    self.recipientReceiver.balance > before(self.recipientReceiver.balance):
        "Recipient balance did not increase"

    self.senderVault.balance == before(self.senderVault.balance) - amount:
        "Sender balance incorrect"
}
```

**Using `before()`**:

```cadence
transaction(amount: UFix64) {
    let vault: &{FungibleToken.Vault}

    prepare(signer: auth(BorrowValue) &Account) {
        self.vault = signer.storage
            .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault")
    }

    execute {
        let withdrawn <- self.vault.withdraw(amount: amount)
        // ... use withdrawn vault
        destroy withdrawn
    }

    post {
        self.vault.balance == before(self.vault.balance) - amount:
            "Balance change does not match withdrawal amount"
    }
}
```

## Phase Execution Order

**CRITICAL**: Phases execute in this exact order:

```
prepare → pre → execute → post
```

**If any phase fails**:
- Entire transaction reverts
- All state changes are rolled back
- Transaction fees are still charged
- Error message is returned to caller

## Transaction Parameters

### Parameter Declaration

```cadence
transaction(
    amount: UFix64,
    recipient: Address,
    memo: String?
) {
    // Transaction body
}
```

### Parameter Types

**Allowed parameter types**:
- Primitives: `Int`, `UInt64`, `UFix64`, `Bool`, `String`, `Address`
- Arrays: `[String]`, `[Address]`, `[UFix64]`
- Dictionaries: `{String: UFix64}`, `{Address: Bool}`
- Optionals: `String?`, `Address?`
- Structs (with allowed field types)

**NOT allowed**:
- Resources (cannot be passed as parameters)
- Capabilities
- References
- Functions

```cadence
// ✅ CORRECT: Valid parameters
transaction(
    amount: UFix64,
    recipients: [Address],
    metadata: {String: String}
) { }

// ❌ WRONG: Invalid parameters
transaction(
    vault: @Vault,  // Resources not allowed
    cap: Capability<&Vault>  // Capabilities not allowed
) { }
```

## Field Declarations

**Declare fields to share state between phases**:

```cadence
transaction(amount: UFix64) {
    // Field declarations
    let senderVault: &{FungibleToken.Provider}
    let recipientVault: &{FungibleToken.Receiver}
    let startBalance: UFix64

    prepare(signer: auth(BorrowValue) &Account) {
        // Initialize fields
        self.senderVault = signer.storage
            .borrow<&{FungibleToken.Provider}>(from: /storage/flowTokenVault)
            ?? panic("Could not borrow FungibleToken Provider reference from /storage/flowTokenVault")

        self.startBalance = self.senderVault.balance
    }

    execute {
        // Use fields
        let vault <- self.senderVault.withdraw(amount: amount)
        self.recipientVault.deposit(from: <-vault)
    }

    post {
        // Reference fields
        self.senderVault.balance == self.startBalance - amount:
            "Balance mismatch"
    }
}
```

## Transaction Imports

**Import contracts/types needed for transaction**:

```cadence
import "FungibleToken"
import "FlowToken"
import "NFTContract"

transaction() {
    // Can use imported types
    let vault: &{FungibleToken.Vault}

    prepare(signer: auth(BorrowValue) &Account) {
        self.vault = signer.storage
            .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault")
    }
}
```

## Best Practices

### Practice 1: Minimal `prepare` Phase

**Keep prepare phase minimal - only account access operations**:

```cadence
// ✅ CORRECT: Minimal prepare
prepare(signer: auth(BorrowValue) &Account) {
    self.vault = signer.storage
        .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault")
}

execute {
    // All business logic here
    let withdrawn <- self.vault.withdraw(amount: amount)
    recipient.deposit(from: <-withdrawn)
}

// ❌ WRONG: Too much in prepare
prepare(signer: auth(BorrowValue) &Account) {
    self.vault = signer.storage
        .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/flowTokenVault")

    // Don't do business logic in prepare
    let withdrawn <- self.vault.withdraw(amount: amount)
    recipient.deposit(from: <-withdrawn)
}
```

### Practice 2: Use `pre` for Validation

**Validate inputs in pre-conditions, not in prepare**:

```cadence
// ✅ CORRECT: Validation in pre
prepare(signer: auth(BorrowValue) &Account) {
    self.vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")
}

pre {
    amount > 0.0: "Amount must be positive"
    amount <= 1000.0: "Amount too large"
}

// ❌ LESS IDEAL: Validation in prepare
prepare(signer: auth(BorrowValue) &Account) {
    assert(amount > 0.0, message: "Amount must be positive")
    assert(amount <= 1000.0, message: "Amount too large")

    self.vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")
}
```

### Practice 3: Use `post` for Invariants

**Document state changes in post-conditions**:

```cadence
post {
    self.vault.balance == before(self.vault.balance) - amount:
        "Balance change incorrect"

    result != nil: "Operation must return a result"
}
```

### Practice 4: Explicit Phase Usage

**Use all relevant phases explicitly** for clarity:

```cadence
// ✅ CORRECT: Explicit phases
transaction(amount: UFix64) {
    let vault: &{FungibleToken.Vault}

    prepare(signer: auth(BorrowValue) &Account) {
        self.vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")
    }

    pre {
        amount > 0.0: "Amount must be positive"
    }

    execute {
        let withdrawn <- self.vault.withdraw(amount: amount)
        destroy withdrawn
    }

    post {
        self.vault.balance >= 0.0: "Balance cannot be negative"
    }
}

// ❌ LESS CLEAR: Everything in prepare
transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")

        assert(amount > 0.0, message: "Amount must be positive")

        let withdrawn <- vault.withdraw(amount: amount)
        destroy withdrawn

        assert(vault.balance >= 0.0, message: "Balance cannot be negative")
    }
}
```

## Required Entitlements

**Specify minimum required entitlements** for account access:

```cadence
// ✅ CORRECT: Specific entitlements
prepare(signer: auth(BorrowValue) &Account) {
    // Only needs BorrowValue
    let vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
}

prepare(signer: auth(BorrowValue, SaveValue) &Account) {
    // Needs both BorrowValue and SaveValue
    let resource <- create MyResource()
    signer.storage.save(<-resource, to: /storage/myResource)
}

prepare(signer: auth(BorrowValue, IssueStorageCapabilityController) &Account) {
    // Needs BorrowValue and capability issuance
    let cap = signer.capabilities.storage
        .issue<&MyResource>(/storage/myResource)
}

// ❌ WRONG: Over-privileged
prepare(signer: auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account) {
    // Only uses BorrowValue but requests everything
    let vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
}
```

### Common Entitlement Combinations

```cadence
// Read-only access
auth(BorrowValue) &Account

// Read and write storage
auth(BorrowValue, SaveValue) &Account

// Storage and capabilities
auth(BorrowValue, SaveValue, IssueStorageCapabilityController) &Account

// Full storage and capability management
auth(BorrowValue, SaveValue, IssueStorageCapabilityController, PublishCapability) &Account

// Contract deployment
auth(BorrowValue, SaveValue, Contracts) &Account

// Key management
auth(BorrowValue, Keys) &Account
```

## Security Considerations

### Consideration 1: Principle of Least Privilege

**Request only necessary entitlements**:

```cadence
// ✅ CORRECT: Minimal entitlements
prepare(signer: auth(BorrowValue) &Account) {
    self.vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")
}

// ❌ AVOID: Unnecessary entitlements
prepare(signer: auth(Storage, Contracts, Keys) &Account) {
    // Only uses BorrowValue
    self.vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
        ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")
}
```

### Consideration 2: Validate External Data

**Never trust external addresses or data without validation**:

```cadence
// ✅ CORRECT: Validate before use
prepare(signer: auth(BorrowValue) &Account) {
    self.recipientCap = getAccount(recipient)
        .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver)

    // Verify capability exists
    if self.recipientCap == nil {
        panic("Could not borrow FungibleToken Receiver reference from /public/receiver for account \(recipient)")
    }
}

// ❌ WRONG: No validation
prepare(signer: auth(BorrowValue) &Account) {
    self.recipientCap = getAccount(recipient)
        .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver)!
    // Force unwrap could panic unexpectedly
}
```

### Consideration 3: Resource Safety

**Always handle resources properly in transactions**:

```cadence
// ✅ CORRECT: Complete resource handling
execute {
    let vault <- self.provider.withdraw(amount: amount)
    self.receiver.deposit(from: <-vault)
    // Resource fully transferred
}

// ❌ WRONG: Resource could be lost
execute {
    let vault <- self.provider.withdraw(amount: amount)
    // If panic happens here, vault is lost!
    someRiskyOperation()
    self.receiver.deposit(from: <-vault)
}

// ✅ BETTER: Defensive resource handling
execute {
    let vault <- self.provider.withdraw(amount: amount)

    // Use defer or ensure all paths handle resource
    if !self.receiver.isValid() {
        destroy vault  // Handle explicitly
        panic("Invalid receiver")
    }

    self.receiver.deposit(from: <-vault)
}
```

## AI Agent Generation Rules

### Rule 1: Always Use Explicit Phases

**Generate transactions with explicit phase separation**:

```cadence
// ✅ CORRECT: Explicit phases
transaction(amount: UFix64) {
    let vault: &{FungibleToken.Vault}

    prepare(signer: auth(BorrowValue) &Account) {
        self.vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")
    }

    pre {
        amount > 0.0: "Amount must be positive"
    }

    execute {
        let withdrawn <- self.vault.withdraw(amount: amount)
        destroy withdrawn
    }

    post {
        self.vault.balance >= 0.0: "Balance non-negative"
    }
}
```

### Rule 2: Minimal Entitlements

**Generate prepare blocks with minimal required entitlements**:

```cadence
// ✅ CORRECT: Only BorrowValue needed
prepare(signer: auth(BorrowValue) &Account) {
    // Borrow operation only
}

// ✅ CORRECT: BorrowValue and SaveValue needed
prepare(signer: auth(BorrowValue, SaveValue) &Account) {
    // Both borrow and save operations
}
```

### Rule 3: Validate Inputs in `pre`

**Generate pre-conditions for parameter validation**:

```cadence
pre {
    amount > 0.0: "Amount must be positive"
    recipient != signer.address: "Cannot send to yourself"
}
```

### Rule 4: Document Invariants in `post`

**Generate post-conditions to document expected outcomes**:

```cadence
post {
    result != nil: "Transaction must produce result"
    self.balance >= 0.0: "Balance cannot be negative"
}
```

## Testing Transactions

```cadence
import Test

access(all) fun testTokenTransfer() {
    let sender = Test.createAccount()
    let recipient = Test.createAccount()

    // Setup accounts
    setupVault(account: sender)
    setupVault(account: recipient)

    // Execute transaction
    let result = executeTransaction(
        "transfer_tokens.cdc",
        [100.0 as UFix64, recipient.address],
        sender
    )

    // Verify success
    Test.expect(result.status, Test.equal(Test.ResultStatus.succeeded))

    // Verify balances
    let recipientBalance = getBalance(account: recipient)
    Test.expect(recipientBalance, Test.equal(100.0))
}
```

## Common Transaction Patterns

### Pattern 1: Token Transfer

```cadence
import "FungibleToken"

transaction(amount: UFix64, to: Address) {
    let sentVault: @{FungibleToken.Vault}

    prepare(signer: auth(BorrowValue) &Account) {
        let vaultRef = signer.storage
            .borrow<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(from: /storage/vault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")

        self.sentVault <- vaultRef.withdraw(amount: amount)
    }

    execute {
        let recipient = getAccount(to)
            .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver)
            ?? panic("Could not borrow FungibleToken Receiver reference from /public/receiver")

        recipient.deposit(from: <-self.sentVault)
    }
}
```

### Pattern 2: Resource Setup

```cadence
import "MyContract"

transaction() {
    prepare(signer: auth(SaveValue, IssueStorageCapabilityController, PublishCapability) &Account) {
        // Create resource
        let resource <- MyContract.createResource()

        // Save to storage
        signer.storage.save(<-resource, to: /storage/myResource)

        // Create and publish capability
        let cap = signer.capabilities.storage
            .issue<&MyContract.Resource>(/storage/myResource)
        signer.capabilities.publish(cap, at: /public/myResource)
    }
}
```

### Pattern 3: Multi-Signature Transaction

```cadence
import "FungibleToken"

transaction(amount: UFix64) {
    prepare(
        sender: auth(BorrowValue) &Account,
        receiver: auth(SaveValue) &Account
    ) {
        // Sender withdraws
        let vault <- sender.storage
            .borrow<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(from: /storage/vault)!
            .withdraw(amount: amount)

        // Receiver saves
        receiver.storage.save(<-vault, to: /storage/receivedVault)
    }
}
```

## Reference Links

- [Cadence Transactions Documentation](https://cadence-lang.org/docs/language/transactions)
- [Account Entitlements](https://cadence-lang.org/docs/language/accounts)
- [Pre and Post Conditions](https://cadence-lang.org/docs/language/pre-and-post-conditions)
