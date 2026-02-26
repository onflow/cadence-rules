# Cadence Pre and Post Conditions Rules

> **Purpose**: Define comprehensive guidelines for using pre-conditions and post-conditions in Cadence. Conditions provide runtime validation and contract enforcement for functions and transactions.

## Core Philosophy: Validate Inputs, Verify Outputs

Pre and post conditions enforce **contracts** between callers and implementations:
- **Pre-conditions**: Verify inputs and preconditions before execution
- **Post-conditions**: Verify outputs and postconditions after execution
- Both cause complete reversion if violated

## Pre-Conditions

### Purpose

Pre-conditions validate that:
- Input parameters are valid
- Required resources exist
- Preconditions are met before execution begins

### Syntax

```cadence
access(all) fun myFunction(amount: UFix64) {
    pre {
        amount > 0.0: "Amount must be positive"
        amount <= 1000.0: "Amount exceeds maximum"
    }

    // Function body
}
```

### Multiple Pre-Conditions

Each condition is a boolean expression with an optional error message:

```cadence
access(all) fun transfer(amount: UFix64, to: Address) {
    pre {
        amount > 0.0: "Amount must be positive"
        amount <= self.balance: "Insufficient balance"
        to != self.owner?.address: "Cannot transfer to self"
    }

    // Function body
}
```

### Pre-Condition Evaluation

- Evaluated **before** function body executes
- All conditions must be `true`
- If any condition is `false`, execution stops and reverts
- Error message is returned to caller

## Post-Conditions

### Purpose

Post-conditions validate that:
- Function produced expected results
- Invariants are maintained
- State changes are correct

### Syntax

```cadence
access(Withdraw) fun withdraw(amount: UFix64): @Vault {
    post {
        result.balance == amount: "Withdrawn amount incorrect"
        self.balance == before(self.balance) - amount: "Balance not updated"
    }

    // Function body
    self.balance = self.balance - amount
    return <- create Vault(balance: amount)
}
```

### The `result` Constant

In post-conditions, `result` refers to the function's return value:

```cadence
access(all) view fun calculateTotal(values: [UFix64]): UFix64 {
    post {
        result >= 0.0: "Total cannot be negative"
        result <= UFix64.max: "Total overflow"
    }

    var total: UFix64 = 0.0
    for value in values {
        total = total + value
    }
    return total
}
```

### The `before()` Function

`before()` captures values from before function execution:

```cadence
access(all) fun increment() {
    post {
        self.counter == before(self.counter) + 1:
            "Counter not incremented correctly"
    }

    self.counter = self.counter + 1
}
```

```cadence
access(all) fun deposit(from: @Vault) {
    let amountDeposited = from.balance

    post {
        self.balance == before(self.balance) + amountDeposited:
            "Balance not updated correctly"
    }

    self.balance = self.balance + from.balance
    destroy from
}
```

## Conditions in Functions

### Basic Function Conditions

```cadence
access(all) entitlement Withdraw

access(all) resource Vault {
    access(self) var balance: UFix64

    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
        pre {
            amount > 0.0: "Amount must be positive"
            amount <= self.balance: "Insufficient balance: available \(self.balance), required \(amount)"
        }

        post {
            result.balance == amount: "Incorrect withdrawal amount"
            self.balance == before(self.balance) - amount: "Balance not updated"
        }

        self.balance = self.balance - amount
        return <- create Vault(balance: amount)
    }

    init(balance: UFix64) {
        self.balance = balance
    }
}
```

### Conditions with Multiple Checks

```cadence
access(all) fun processTransaction(
    from: &Vault,
    to: &Vault,
    amount: UFix64
) {
    pre {
        amount > 0.0: "Amount must be positive"
        from.balance >= amount: "Sender has insufficient balance"
        to.canReceive(): "Receiver cannot accept funds"
    }

    post {
        from.balance == before(from.balance) - amount:
            "Sender balance incorrect"
        to.balance == before(to.balance) + amount:
            "Receiver balance incorrect"
    }

    let withdrawn <- from.withdraw(amount: amount)
    to.deposit(from: <-withdrawn)
}
```

## Conditions in Transactions

### Transaction Pre-Conditions

Execute **after** `prepare`, **before** `execute`:

```cadence
transaction(amount: UFix64, recipient: Address) {
    let vault: &Vault

    prepare(signer: auth(BorrowValue) &Account) {
        self.vault = signer.storage.borrow<&Vault>(from: /storage/vault)
            ?? panic("Could not borrow Vault reference from /storage/vault")
    }

    pre {
        amount > 0.0: "Amount must be positive"
        amount <= self.vault.balance: "Insufficient balance: available \(self.vault.balance), required \(amount)"
        recipient != signer.address: "Cannot send to yourself"
    }

    execute {
        // Transaction logic
    }
}
```

### Transaction Post-Conditions

Execute **after** `execute`:

```cadence
transaction(amount: UFix64) {
    let vault: &Vault
    let startBalance: UFix64

    prepare(signer: auth(BorrowValue) &Account) {
        self.vault = signer.storage.borrow<&Vault>(from: /storage/vault)
            ?? panic("Could not borrow Vault reference from /storage/vault")
        self.startBalance = self.vault.balance
    }

    execute {
        let withdrawn <- self.vault.withdraw(amount: amount)
        // ... use withdrawn vault
        destroy withdrawn
    }

    post {
        self.vault.balance == self.startBalance - amount:
            "Balance not updated correctly"
        self.vault.balance >= 0.0:
            "Balance cannot be negative"
    }
}
```

### Complete Transaction Example

```cadence
transaction(amount: UFix64, recipient: Address) {
    let senderVault: &Vault
    let recipientVault: &{VaultPublic}
    let startSenderBalance: UFix64
    let startRecipientBalance: UFix64

    prepare(signer: auth(BorrowValue) &Account) {
        self.senderVault = signer.storage.borrow<&Vault>(from: /storage/vault)
            ?? panic("Could not borrow Vault reference from /storage/vault")

        self.recipientVault = getAccount(recipient)
            .capabilities.borrow<&{VaultPublic}>(/public/vault)
            ?? panic("Could not borrow VaultPublic reference from /public/vault")

        self.startSenderBalance = self.senderVault.balance
        self.startRecipientBalance = self.recipientVault.balance
    }

    pre {
        amount > 0.0: "Amount must be positive"
        amount <= self.senderVault.balance: "Insufficient balance: available \(self.senderVault.balance), required \(amount)"
        recipient != signer.address: "Cannot send to yourself"
    }

    execute {
        let withdrawn <- self.senderVault.withdraw(amount: amount)
        self.recipientVault.deposit(from: <-withdrawn)
    }

    post {
        self.senderVault.balance == self.startSenderBalance - amount:
            "Sender balance incorrect"
        self.recipientVault.balance == self.startRecipientBalance + amount:
            "Recipient balance incorrect"
    }
}
```

## View Context Restrictions

### Conditions Are View Contexts

**Conditions cannot modify state** - they're read-only:

```cadence
// ❌ WRONG: State modification in condition
access(all) fun myFunction() {
    pre {
        self.counter = self.counter + 1  // COMPILE ERROR: cannot modify state
        self.counter > 0: "Counter must be positive"
    }
}

// ✅ CORRECT: Read-only checks
access(all) fun myFunction() {
    pre {
        self.counter > 0: "Counter must be positive"
        self.isValid(): "Invalid state"
    }

    // Modifications happen in body
    self.counter = self.counter + 1
}
```

### Allowed in Conditions

- Reading fields
- Calling view functions
- Boolean operations
- Comparisons
- `before()` function (post-conditions only)
- `result` constant (post-conditions only)

### Not Allowed in Conditions

- Modifying fields
- Calling non-view functions
- Resource operations
- Emitting events

## Conditions in Interfaces

### Interface Conditions

Interfaces can define conditions that implementations inherit:

```cadence
access(all) entitlement Withdraw

access(all) resource interface Vault {
    access(all) var balance: UFix64

    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
        pre {
            amount > 0.0: "Amount must be positive"
            amount <= self.balance: "Insufficient balance: available \(self.balance), required \(amount)"
        }
        post {
            result.balance == amount: "Incorrect amount"
            self.balance == before(self.balance) - amount: "Balance mismatch"
        }
    }
}

// Implementation inherits conditions
access(all) resource MyVault: Vault {
    access(all) var balance: UFix64

    // Must satisfy interface conditions
    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
        // Interface pre-condition checked automatically
        self.balance = self.balance - amount
        return <- create MyVault(balance: amount)
        // Interface post-condition checked automatically
    }

    init(balance: UFix64) {
        self.balance = balance
    }
}
```

### Condition Inheritance

All interface conditions are automatically checked:

```cadence
let vault <- create MyVault(balance: 100.0)

// Pre-condition from interface checked
// Implementation code runs
// Post-condition from interface checked
let withdrawn <- vault.withdraw(amount: 10.0)

destroy withdrawn
destroy vault
```

### Adding Implementation Conditions

Implementations can add additional conditions:

```cadence
access(all) resource MyVault: Vault {
    access(all) var balance: UFix64
    access(all) var locked: Bool

    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
        // Interface conditions checked
        pre {
            // Additional implementation condition
            !self.locked: "Vault is locked"
        }

        post {
            // Additional implementation condition
            !self.locked: "Vault should not be locked after withdraw"
        }

        self.balance = self.balance - amount
        return <- create MyVault(balance: amount, locked: false)
    }

    init(balance: UFix64, locked: Bool) {
        self.balance = balance
        self.locked = locked
    }
}
```

## Best Practices

### Practice 1: Descriptive Error Messages

**Always include clear error messages.**

```cadence
// ✅ CORRECT: Clear error message
pre {
    amount > 0.0: "Transfer amount must be greater than zero"
    amount <= self.balance: "Insufficient balance: have \(self.balance), need \(amount)"
}

// ❌ LESS HELPFUL: Generic message
pre {
    amount > 0.0: "Invalid amount"
    amount <= self.balance: "Error"
}
```

### Practice 2: Check Inputs in Pre-Conditions

**Validate parameters before processing.**

```cadence
access(all) fun setConfig(config: Config) {
    pre {
        config.isValid(): "Invalid configuration"
        config.timeout > 0: "Timeout must be positive"
        config.maxAttempts <= 10: "Too many retry attempts"
    }

    self.config = config
}
```

### Practice 3: Verify Invariants in Post-Conditions

**Ensure state remains consistent.**

```cadence
access(all) fun transferOwnership(newOwner: Address) {
    post {
        self.owner == newOwner: "Ownership not transferred"
        before(self.owner) != newOwner: "New owner same as old owner"
    }

    self.owner = newOwner
}
```

### Practice 4: Use `before()` for State Changes

**Document expected state changes.**

```cadence
access(all) fun increment(by: UInt64) {
    post {
        self.counter == before(self.counter) + by:
            "Counter not incremented by expected amount"
    }

    self.counter = self.counter + by
}
```

### Practice 5: Single Responsibility per Condition

**One check per condition statement.**

```cadence
// ✅ GOOD: Separate conditions
pre {
    amount > 0.0: "Amount must be positive"
    amount <= maxAmount: "Amount exceeds maximum"
    recipient != sender: "Cannot transfer to self"
}

// ❌ LESS CLEAR: Combined conditions
pre {
    amount > 0.0 && amount <= maxAmount && recipient != sender:
        "Invalid transfer parameters"
}
```

## Common Patterns

### Pattern 1: Balance Validation

```cadence
access(Withdraw) fun withdraw(amount: UFix64): @Vault {
    pre {
        amount > 0.0: "Amount must be positive"
        self.balance >= amount: "Insufficient balance: available \(self.balance), required \(amount)"
    }

    post {
        result.balance == amount: "Withdrawn amount incorrect"
        self.balance == before(self.balance) - amount: "Balance not updated"
    }

    self.balance = self.balance - amount
    return <- create Vault(balance: amount)
}
```

### Pattern 2: State Transition Validation

```cadence
access(all) enum Status: UInt8 {
    access(all) case pending
    access(all) case active
    access(all) case completed
}

access(all) fun complete() {
    pre {
        self.status == Status.active: "Can only complete active items"
    }

    post {
        self.status == Status.completed: "Status not updated to completed"
        before(self.status) == Status.active: "Invalid state transition"
    }

    self.status = Status.completed
}
```

### Pattern 3: Resource Integrity

```cadence
access(all) fun mergeVaults(other: @Vault): @Vault {
    let otherBalance = other.balance

    pre {
        other.balance > 0.0: "Cannot merge empty vault"
    }

    post {
        result.balance == before(self.balance) + otherBalance:
            "Merged balance incorrect"
    }

    let totalBalance = self.balance + other.balance
    destroy other

    return <- create Vault(balance: totalBalance)
}
```

### Pattern 4: Array/Dictionary Validation

```cadence
access(all) fun addItem(item: Item) {
    pre {
        !self.items.contains(item.id): "Item already exists"
        self.items.length < self.maxItems: "Collection full"
    }

    post {
        self.items.contains(item.id): "Item not added"
        self.items.length == before(self.items.length) + 1:
            "Collection size not updated"
    }

    self.items.append(item)
}
```

## AI Agent Generation Rules

### Rule 1: Generate Pre-Conditions for Input Validation

**Always validate function parameters.**

```cadence
access(all) fun transfer(amount: UFix64, to: Address) {
    pre {
        amount > 0.0: "Amount must be positive"
        to != self.owner?.address: "Cannot transfer to self"
    }

    // Function body
}
```

### Rule 2: Generate Post-Conditions for Critical Operations

**Verify important state changes.**

```cadence
access(Withdraw) fun withdraw(amount: UFix64): @Vault {
    post {
        result.balance == amount: "Incorrect withdrawal"
        self.balance == before(self.balance) - amount: "Balance mismatch"
    }

    // Function body
}
```

### Rule 3: Use Descriptive Error Messages

**Generate clear, actionable error messages.**

```cadence
pre {
    amount > 0.0: "Transfer amount must be greater than zero"
    amount <= self.balance: "Insufficient balance: have \(self.balance), need \(amount)"
}
```

### Rule 4: Generate `before()` for State Tracking

**Track state changes in post-conditions.**

```cadence
post {
    self.counter == before(self.counter) + 1: "Counter not incremented"
    self.balance >= before(self.balance): "Balance decreased unexpectedly"
}
```

## Testing Conditions

```cadence
import Test

access(all) fun testPreConditionSuccess() {
    let vault <- createVault(balance: 100.0)

    // Valid withdrawal - pre-condition passes
    let withdrawn <- vault.withdraw(amount: 10.0)

    Test.expect(withdrawn.balance, Test.equal(10.0))

    destroy withdrawn
    destroy vault
}

access(all) fun testPreConditionFailure() {
    let vault <- createVault(balance: 100.0)

    // Invalid withdrawal - pre-condition fails
    Test.expectFailure(fun() {
        let withdrawn <- vault.withdraw(amount: 200.0)
        destroy withdrawn
    }, errorMessageContains: "Insufficient balance")

    destroy vault
}

access(all) fun testPostCondition() {
    let vault <- createVault(balance: 100.0)
    let startBalance = vault.balance

    let withdrawn <- vault.withdraw(amount: 10.0)

    // Post-condition verified
    Test.expect(
        vault.balance,
        Test.equal(startBalance - 10.0)
    )

    destroy withdrawn
    destroy vault
}
```

## Reference Links

- [Cadence Pre and Post Conditions Documentation](https://cadence-lang.org/docs/language/pre-and-post-conditions)
- [Functions Documentation](https://cadence-lang.org/docs/language/functions)
- [Transactions Documentation](https://cadence-lang.org/docs/language/transactions)
- [View Functions](https://cadence-lang.org/docs/language/functions#view-functions)
