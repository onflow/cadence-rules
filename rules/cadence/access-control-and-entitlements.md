# Cadence Access Control and Entitlements Rules

> **Purpose**: Define security-first guidelines for access control modifiers and entitlements in Cadence code generation. These rules prioritize secure defaults and explicit access declarations.

## Core Philosophy: Secure by Default

**Default to Private, Explicitly Grant Access**

All fields and functions MUST start with restrictive access and only be made more permissive when explicitly required. This prevents unintentional exposure of privileged operations.

## Access Modifier Hierarchy (Most to Least Restrictive)

1. `access(self)` - Private to current scope only
2. `access(contract)` - Visible within declaring contract only
3. `access(account)` - Visible to all contracts in the same account
4. `access(Entitlement)` - Entitled access with specific permissions
5. `access(all)` - Public access (use sparingly)

## Mandatory Access Control Rules

### Rule 1: Default Access Levels

**Fields (Variables)**
- **Default**: `access(self)` or `access(contract)`
- Storage values and internal state MUST be private by default
- Only expose fields when absolutely necessary for external access

```cadence
// ✅ CORRECT: Private by default
access(contract) var balance: UFix64
access(self) let id: UInt64

// ❌ WRONG: Unnecessary public exposure
access(all) var balance: UFix64
```

**Functions**
- **Default**: `access(self)` or `access(contract)`
- Start restrictive, then add entitlements for privileged operations
- Only use `access(all)` for explicitly public operations

```cadence
// ✅ CORRECT: Private internal function
access(self) fun updateInternalState() {
    // Implementation
}

// ✅ CORRECT: Contract-level helper
access(contract) fun calculateFee(amount: UFix64): UFix64 {
    return amount * 0.01
}

// ❌ WRONG: Default public access
access(all) fun updateInternalState() {
    // Unintentionally exposed
}
```

### Rule 2: When to Use `access(all)`

Only use `access(all)` for:

1. **View Functions** - Read-only operations that don't modify state
2. **Public Interfaces** - Functions explicitly designed for external calls (e.g., `deposit`, `transfer`)
3. **Public Getters** - Explicit data accessors

```cadence
// ✅ CORRECT: Public view function
access(all) view fun getBalance(): UFix64 {
    return self.balance
}

// ✅ CORRECT: Public deposit (designed for anyone to call)
access(all) fun deposit(from: @{FungibleToken.Vault}) {
    // Safe public operation
}

// ❌ WRONG: Administrative function made public
access(all) fun setBalance(newBalance: UFix64) {
    self.balance = newBalance  // Should require authorization!
}
```

### Rule 3: Entitlements for Privileged Access

**When to Use Entitlements**:
- Administrative functions (owner/admin only operations)
- State-modifying privileged operations
- Resource management functions requiring authorization
- Mixed-access resources (both public and privileged functions)

**Entitlement Declaration Pattern**:
```cadence
// Define entitlements at contract level
access(all) entitlement Admin
access(all) entitlement Withdraw
access(all) entitlement Mint

// Use entitlements on functions
access(Admin) fun updateSettings(newConfig: Config) {
    // Only accessible with Admin entitlement
}

access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
    // Only accessible with Withdraw entitlement
}
```

### Rule 4: Resource Access Patterns

**Mixed-Access Resources**

Resources that have both public and privileged operations MUST use entitlements to separate access levels:

```cadence
// ✅ CORRECT: Mixed-access resource with entitlements
access(all) resource Vault {
    // Private storage
    access(self) var balance: UFix64

    // Public view function
    access(all) view fun getBalance(): UFix64 {
        return self.balance
    }

    // Public deposit (anyone can call)
    access(all) fun deposit(from: @{FungibleToken.Vault}) {
        self.balance = self.balance + from.balance
        destroy from
    }

    // Entitled withdraw (requires authorization)
    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        pre {
            self.balance >= amount: "Insufficient balance"
        }
        self.balance = self.balance - amount
        return <-create Vault(balance: amount)
    }

    // Entitled admin function
    access(Admin) fun forceSetBalance(newBalance: UFix64) {
        self.balance = newBalance
    }
}
```

**Separate Resources Pattern**

When operations are clearly separated, consider using distinct resources:

```cadence
// Public interface resource
access(all) resource VaultPublic {
    access(all) view fun getBalance(): UFix64
    access(all) fun deposit(from: @{FungibleToken.Vault})
}

// Private admin resource (stored separately)
access(all) resource VaultAdmin {
    access(all) fun withdraw(amount: UFix64): @{FungibleToken.Vault}
    access(all) fun setBalance(newBalance: UFix64)
}
```

### Rule 5: Built-in Mutability Entitlements

Use standard entitlements for collection operations:
- `Insert` - Add elements
- `Remove` - Delete elements
- `Mutate` - Both insert and remove (equivalent to `Insert, Remove`)

```cadence
access(all) resource Collection {
    access(self) var items: @{UInt64: NFT}

    // Requires Insert entitlement
    access(Insert) fun addItem(_ item: @NFT) {
        let id = item.id
        self.items[id] <-! item
    }

    // Requires Remove entitlement
    access(Remove) fun removeItem(id: UInt64): @NFT {
        return <-self.items.remove(key: id)!
    }

    // Requires both (Mutate entitlement)
    access(Mutate) fun replaceItem(id: UInt64, with: @NFT): @NFT {
        return <-self.items.insert(key: id, <-with)
    }
}
```

## Entitlement Combinators

### Conjunction (`,`) - Requires ALL entitlements
```cadence
access(Admin, Withdraw) fun adminWithdraw(amount: UFix64): @{FungibleToken.Vault} {
    // Requires BOTH Admin AND Withdraw entitlements
}
```

### Disjunction (`|`) - Requires ANY entitlement
```cadence
access(Admin | Owner) fun privilegedAction() {
    // Requires EITHER Admin OR Owner entitlement
}
```

## Critical Security Warnings

### ⚠️ Warning 1: Public Functions Are Completely Open

Making a function `access(all)` means **anyone** can call it:

```cadence
// ❌ CRITICAL SECURITY FLAW
access(all) resource Vault {
    access(all) var balance: UFix64

    // Anyone can drain this vault!
    access(all) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        self.balance = self.balance - amount
        return <-create Vault(balance: amount)
    }
}

// ✅ CORRECT: Use entitlements
access(all) resource Vault {
    access(self) var balance: UFix64

    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        self.balance = self.balance - amount
        return <-create Vault(balance: amount)
    }
}
```

### ⚠️ Warning 2: Private ≠ Secret

`access(self)` controls **programmatic access**, NOT data visibility. All account storage is publicly readable on-chain:

```cadence
// ❌ WRONG: Storing secrets
access(all) resource User {
    access(self) let secretKey: String  // NOT SECURE - visible on chain!
    access(self) let password: String   // NOT SECURE - visible on chain!
}
```

**Never store secrets, passwords, or private keys in account storage.**

### ⚠️ Warning 3: Accidental Exposure in Capabilities

When creating capabilities, only expose authorized reference types:

```cadence
// ❌ WRONG: Exposing full access
let vaultCap = account.capabilities.storage
    .issue<&Vault>(/storage/vault)  // Full access!
account.capabilities.publish(vaultCap, at: /public/vault)

// ✅ CORRECT: Expose restricted reference
let vaultCap = account.capabilities.storage
    .issue<&{VaultPublic}>(/storage/vault)  // Only public interface
account.capabilities.publish(vaultCap, at: /public/vault)

// ✅ ALSO CORRECT: Use entitled reference for privileged access
let adminCap = account.capabilities.storage
    .issue<auth(Admin, Withdraw) &Vault>(/storage/vault)
// Store privately or give to trusted party
```

## AI Agent Generation Guidelines

### Priority 1: Always Start Restrictive

When generating new code:
1. Default ALL fields to `access(self)` or `access(contract)`
2. Default ALL functions to `access(self)` or `access(contract)`
3. Only make functions `access(all)` if they are:
   - View functions (read-only)
   - Explicitly designed for public access (deposit, transfer)
   - Public getters

### Priority 2: Use Entitlements Liberally

**It is better to over-use entitlements than under-use them.**

Entitlements are inherently more secure. If an AI generates code with unnecessary entitlements, the developer can adjust. But if the AI generates code without necessary entitlements, it creates security vulnerabilities.

**Entitlement Application Strategy**:
- **Admin operations**: Always use `access(Admin)` or similar
- **State modifications**: Consider `access(Mutate)` or custom entitlements
- **Resource withdrawal**: Always use `access(Withdraw)` or similar
- **Mixed-access resources**: Use multiple entitlements to separate public and privileged operations

```cadence
// ✅ PREFERRED: Liberal entitlement use
access(all) resource Manager {
    access(self) var data: String

    access(all) view fun getData(): String {
        return self.data
    }

    access(Update) fun updateData(newData: String) {
        self.data = newData
    }

    access(Admin) fun resetData() {
        self.data = ""
    }
}

// ❌ AVOID: No access control on privileged operations
access(all) resource Manager {
    access(all) var data: String

    access(all) fun updateData(newData: String) {
        self.data = newData
    }

    access(all) fun resetData() {
        self.data = ""
    }
}
```

### Priority 3: Explicit Developer Choice

The goal is to **force developers to make explicit security decisions**:

- AI generates restrictive code by default
- Developer must explicitly relax restrictions if needed
- Developer must read about entitlements to understand the code
- Developer learns secure patterns through exposure

## Common Patterns

### Pattern 1: Public View, Entitled Mutate

```cadence
access(all) resource Counter {
    access(self) var count: UInt64

    access(all) view fun getCount(): UInt64 {
        return self.count
    }

    access(Increment) fun increment() {
        self.count = self.count + 1
    }

    access(Admin) fun reset() {
        self.count = 0
    }
}
```

### Pattern 2: Interface with Entitlements

```cadence
access(all) entitlement Withdraw
access(all) entitlement Deposit

access(all) resource interface VaultInterface {
    access(all) view fun getBalance(): UFix64
    access(Deposit) fun deposit(from: @{FungibleToken.Vault})
    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault}
}
```

### Pattern 3: Admin Resource with Multiple Entitlements

```cadence
access(all) entitlement Admin
access(all) entitlement Configure
access(all) entitlement Pause

access(all) resource ProtocolManager {
    access(self) var isPaused: Bool
    access(self) var config: Config

    access(all) view fun isPaused(): Bool {
        return self.isPaused
    }

    access(Pause) fun pause() {
        self.isPaused = true
    }

    access(Pause) fun unpause() {
        self.isPaused = false
    }

    access(Configure) fun updateConfig(newConfig: Config) {
        self.config = newConfig
    }

    access(Admin) fun emergencyShutdown() {
        // Critical admin function
    }
}
```

## Testing Checklist

When reviewing generated code:

- [ ] All fields use `access(self)` or `access(contract)` unless explicitly needed
- [ ] All functions use `access(self)` or `access(contract)` by default
- [ ] View functions use `access(all)` appropriately
- [ ] Privileged operations use entitlements
- [ ] Admin functions use `access(Admin)` or similar entitlement
- [ ] Mixed-access resources separate public and privileged operations
- [ ] No secrets stored in any access level fields
- [ ] Capabilities only expose intended reference types
- [ ] Entitlements are defined at contract level before use

## Reference Links

- [Cadence Access Control Documentation](https://cadence-lang.org/docs/language/access-control)
- [Cadence Capabilities Documentation](https://cadence-lang.org/docs/language/capabilities)
