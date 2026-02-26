# Cadence Security Best Practices

> **Purpose**: Comprehensive security guidelines for Cadence development. These practices prevent common vulnerabilities and ensure safe smart contract implementation.

## Core Security Philosophy

**Secure by Default, Explicit by Design**

Cadence's security model is built on:
1. **Resource-oriented programming** - Assets are resources that can't be copied or lost
2. **Capability-based security** - Access is granted through unforgeable capabilities
3. **Explicit access control** - All permissions must be declared explicitly
4. **Type safety** - Strong typing prevents many classes of vulnerabilities

## Access Control Security

### Practice 1: Default to Private Access

**Start with `access(self)`, expand only when necessary.**

```cadence
// ✅ CORRECT: Restrictive by default
access(all) contract MyContract {
    // Private internal state
    access(self) var internalCounter: UInt64
    access(self) let adminAddress: Address

    // Contract-level helper
    access(contract) fun validateState(): Bool {
        return self.internalCounter > 0
    }

    // Public view function
    access(all) view fun getCounter(): UInt64 {
        return self.internalCounter
    }

    // Entitled admin function
    access(Admin) fun resetCounter() {
        self.internalCounter = 0
    }
}

// ❌ WRONG: Unnecessarily public
access(all) contract MyContract {
    access(all) var internalCounter: UInt64  // Anyone can read
    access(all) let adminAddress: Address    // Admin address exposed

    access(all) fun validateState(): Bool {  // Internal helper exposed
        return self.internalCounter > 0
    }

    access(all) fun resetCounter() {  // No access control!
        self.internalCounter = 0
    }
}
```

**Rule**: Only use `access(all)` for:
- Public view functions (read-only data)
- Intentionally public APIs (deposit, transfer)
- Interface implementations meant for external use

### Practice 2: Protect Composite-Typed Fields

**Resources, structs, and capabilities stored as fields MUST be `access(self)`.**

Even if you prevent field reassignment with `access(all)`, the underlying object remains mutable through its functions.

```cadence
// ❌ CRITICAL VULNERABILITY: Public capability field
access(all) contract VulnerableContract {
    // Anyone can copy this capability!
    access(all) let adminCapability: Capability<&Admin>

    init(adminCap: Capability<&Admin>) {
        self.adminCapability = adminCap
    }
}

// Attacker code:
let stolenCap = VulnerableContract.adminCapability
// Now attacker has admin access!

// ✅ CORRECT: Private capability field
access(all) contract SecureContract {
    access(self) let adminCapability: Capability<&Admin>

    init(adminCap: Capability<&Admin>) {
        self.adminCapability = adminCap
    }

    // Expose functionality, not the capability
    access(Admin) fun performAdminAction() {
        if let admin = self.adminCapability.borrow() {
            admin.doSomething()
        }
    }
}
```

**Critical Types to Protect**:
- Resources (`@Resource`)
- Capabilities (`Capability<T>`)
- Structs with mutable state
- Arrays and dictionaries of the above

### Practice 3: Use Entitlements for Privileged Operations

**Never use `access(all)` for state-modifying or privileged operations.**

```cadence
access(all) entitlement Owner
access(all) entitlement Withdraw
access(all) entitlement Admin

access(all) resource Vault {
    access(self) var balance: UFix64

    // Public view - safe
    access(all) view fun getBalance(): UFix64 {
        return self.balance
    }

    // Public deposit - safe (adding value)
    access(all) fun deposit(from: @{FungibleToken.Vault}) {
        self.balance = self.balance + from.balance
        destroy from
    }

    // Entitled withdraw - requires authorization
    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        pre {
            self.balance >= amount: "Insufficient balance: available \(self.balance), required \(amount)"
        }
        self.balance = self.balance - amount
        return <- create Vault(balance: amount)
    }

    // Entitled admin function - requires authorization
    access(Admin) fun setBalance(newBalance: UFix64) {
        self.balance = newBalance
    }
}
```

## Capability Security

### Practice 4: Issue Capabilities Sparingly

**Only create capabilities when truly necessary.**

```cadence
// ✅ CORRECT: Check before issuing
prepare(signer: auth(IssueStorageCapabilityController, Capabilities) &Account) {
    // Check if capability already exists
    let existingControllers = signer.capabilities.storage
        .getControllers(forPath: /storage/vault)

    if existingControllers.length == 0 {
        // Only issue if none exist
        let controller = signer.capabilities.storage
            .issue<&Vault>(/storage/vault)

        controller.setTag("Public receiver capability")
    }
}

// ❌ WRONG: Blindly issuing without checking
prepare(signer: auth(IssueStorageCapabilityController) &Account) {
    // Creates duplicate capabilities wastefully
    let controller = signer.capabilities.storage
        .issue<&Vault>(/storage/vault)
}
```

### Practice 5: Publish with Verification

**Check for existing published capabilities before publishing new ones.**

```cadence
// ✅ CORRECT: Verify before publishing
prepare(signer: auth(IssueStorageCapabilityController, PublishCapability) &Account) {
    // Check if already published
    let existing = signer.capabilities
        .borrow<&{FungibleToken.Receiver}>(/public/receiver)

    if existing == nil {
        // Only publish if doesn't exist
        let controller = signer.capabilities.storage
            .issue<&Vault>(/storage/vault)

        let cap = controller.capability
        signer.capabilities.publish(cap, at: /public/receiver)
    }
}

// ❌ WRONG: Publishing without checking
prepare(signer: auth(IssueStorageCapabilityController, PublishCapability) &Account) {
    let controller = signer.capabilities.storage
        .issue<&Vault>(/storage/vault)

    let cap = controller.capability
    signer.capabilities.publish(cap, at: /public/receiver)
    // May overwrite existing capability
}
```

### Practice 6: Validate Before Borrowing

**Always use `check()` or handle optional returns when borrowing capabilities.**

```cadence
// ✅ CORRECT: Check before borrowing
let cap = getAccount(address)
    .capabilities.get<&{FungibleToken.Receiver}>(/public/receiver)

if cap.check() {
    if let receiver = cap.borrow() {
        receiver.deposit(from: <-vault)
    } else {
        // Handle borrow failure
        destroy vault
    }
} else {
    // Handle invalid capability
    destroy vault
}

// ✅ ALSO CORRECT: Handle optional directly
if let receiver = getAccount(address)
    .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver) {
    receiver.deposit(from: <-vault)
} else {
    destroy vault
}

// ❌ WRONG: Force unwrap without checking
let receiver = getAccount(address)
    .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver)!
receiver.deposit(from: <-vault)  // Panics if capability invalid
```

### Practice 7: Never Expose Capabilities in Public Fields

**Capabilities in public fields can be copied by anyone.**

```cadence
// ❌ CRITICAL VULNERABILITY: Public capability field
access(all) resource Collection {
    access(all) let adminCap: Capability<auth(Admin) &Admin>
    // Anyone can copy this!

    init(adminCap: Capability<auth(Admin) &Admin>) {
        self.adminCap = adminCap
    }
}

// ✅ CORRECT: Private capability field
access(all) resource Collection {
    access(self) let adminCap: Capability<auth(Admin) &Admin>

    init(adminCap: Capability<auth(Admin) &Admin>) {
        self.adminCap = adminCap
    }

    // Expose functionality, not capability
    access(all) fun performAdminAction() {
        if let admin = self.adminCap.borrow() {
            admin.doSomething()
        }
    }
}

// ✅ BEST: Store capability privately, expose through controlled interface
access(all) resource Collection {
    access(self) let adminCap: Capability<auth(Admin) &Admin>

    init(adminCap: Capability<auth(Admin) &Admin>) {
        self.adminCap = adminCap
    }
}

// Admin operations performed through entitled functions
```

### Practice 8: Never Expose Capabilities in Public Arrays/Dictionaries

**Collections of capabilities are equally vulnerable.**

```cadence
// ❌ WRONG: Public array of capabilities
access(all) contract VulnerableContract {
    access(all) let capabilities: [Capability<&Admin>]
    // Anyone can access all capabilities!

    init() {
        self.capabilities = []
    }
}

// ✅ CORRECT: Private storage
access(all) contract SecureContract {
    access(self) let capabilities: {Address: Capability<&Admin>}

    init() {
        self.capabilities = {}
    }

    access(Admin) fun getCapability(for: Address): Capability<&Admin>? {
        return self.capabilities[for]
    }
}
```

## Reference Security

### Practice 9: References Cannot Be Stored

**Use capabilities for persistent access, not references.**

```cadence
// ❌ WRONG: Attempting to store reference
access(all) resource Container {
    access(self) let vaultRef: &{FungibleToken.Vault}  // COMPILE ERROR

    init(vaultRef: &{FungibleToken.Vault}) {
        self.vaultRef = vaultRef
    }
}

// ✅ CORRECT: Store capability instead
access(all) resource Container {
    access(self) let vaultCap: Capability<&{FungibleToken.Vault}>

    init(vaultCap: Capability<&{FungibleToken.Vault}>) {
        self.vaultCap = vaultCap
    }

    access(all) fun useVault() {
        if let vault = self.vaultCap.borrow() {
            // Use vault reference
        }
    }
}
```

### Practice 10: Minimize Entitlements on References

**Grant only necessary entitlements when creating authorized references.**

```cadence
// ✅ CORRECT: Minimal entitlements
let vaultRef = account.storage
    .borrow<auth(FungibleToken.Withdraw) &Vault>(from: /storage/vault)
// Only Withdraw entitlement

// ❌ WRONG: Over-privileged
let vaultRef = account.storage
    .borrow<auth(FungibleToken.Withdraw, Owner, Admin, Mutate) &Vault>(from: /storage/vault)
// Unnecessary entitlements
```

## Account Security

### Practice 11: Never Trust User Storage

**Users control their own storage completely.**

```cadence
// ❌ WRONG: Assuming type without validation
access(all) fun processUserData(userAddress: Address) {
    let data = getAccount(userAddress).storage
        .borrow<&MyData>(from: /storage/myData)!
    // User could store anything at this path!

    data.process()  // May not be MyData!
}

// ✅ CORRECT: Validate type before use
access(all) fun processUserData(userAddress: Address) {
    if let data = getAccount(userAddress).storage
        .borrow<&MyData>(from: /storage/myData) {

        // Verify it's actually MyData
        if data.getType() == Type<@MyData>() {
            data.process()
        } else {
            log("Invalid data type at path")
        }
    }
}

// ✅ BEST: Use capabilities with explicit types
access(all) fun processUserData(userAddress: Address) {
    // Capability type ensures correct type
    if let data = getAccount(userAddress)
        .capabilities.borrow<&{MyDataPublic}>(/public/myData) {
        data.process()
    }
}
```

### Practice 12: Avoid Passing Authorized Account References

**Never pass `auth(...) &Account` to functions unless absolutely necessary.**

```cadence
// ❌ DANGEROUS: Function receives full account access
access(all) fun dangerousFunction(account: auth(Storage, Contracts) &Account) {
    // This function can do ANYTHING with the account
    // Could deploy malicious contracts
    // Could steal all resources
}

// ✅ BETTER: Pass specific capabilities
access(all) fun safeFunction(vaultCap: Capability<auth(Withdraw) &Vault>) {
    // Only has access to withdraw from vault
    // Limited, revocable access
}

// ✅ BEST: Perform operations in transaction prepare block
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // All account access happens here
        let vault = signer.storage.borrow<&Vault>(from: /storage/vault)
            ?? panic("Could not borrow Vault reference from /storage/vault")

        // Pass vault reference to function
        MyContract.safeFunction(vault: vault)
    }
}
```

### Practice 13: Use Minimal Transaction Entitlements

**Request only the entitlements you actually need.**

```cadence
// ✅ CORRECT: Minimal entitlements
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Only needs BorrowValue
        let vault = signer.storage.borrow<&Vault>(from: /storage/vault)
    }
}

// ❌ WRONG: Over-privileged
transaction() {
    prepare(signer: auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account) {
        // Only uses BorrowValue but requests everything!
        let vault = signer.storage.borrow<&Vault>(from: /storage/vault)
    }
}
```

**Common Minimal Entitlement Patterns**:
```cadence
// Read-only
auth(BorrowValue) &Account

// Read and write storage
auth(BorrowValue, SaveValue) &Account

// Capability issuance
auth(BorrowValue, IssueStorageCapabilityController) &Account

// Capability publishing
auth(BorrowValue, IssueStorageCapabilityController, PublishCapability) &Account
```

## Type Safety

### Practice 14: Use Most Specific Types

**Always use the most restrictive, specific types possible.**

```cadence
// ✅ CORRECT: Specific types
access(all) fun processNFT(nft: @MyNFT) {
    // Function expects exact type
    destroy nft
}

access(all) fun processVault(vault: &{FungibleToken.Balance, FungibleToken.Receiver}) {
    // Specific interface combination
}

// ❌ LESS SAFE: Generic types
access(all) fun processNFT(nft: @{NonFungibleToken.NFT}) {
    // Could be any NFT type
    destroy nft
}

access(all) fun processVault(vault: &AnyResource) {
    // Could be anything!
}
```

### Practice 15: Cast Less-Specific Types

**When receiving generic types, cast to expected concrete types.**

```cadence
// ✅ CORRECT: Type verification and casting
access(all) fun processGenericNFT(nft: @{NonFungibleToken.NFT}) {
    // Verify and cast to specific type
    if nft.getType() == Type<@MyNFT>() {
        let specificNFT <- nft as! @MyNFT
        // Now safely use specific type
        specificNFT.specificMethod()
        destroy specificNFT
    } else {
        destroy nft
    }
}

// ❌ WRONG: Using without verification
access(all) fun processGenericNFT(nft: @{NonFungibleToken.NFT}) {
    // Assuming it's MyNFT without checking
    let specificNFT <- nft as! @MyNFT  // May panic!
    specificNFT.specificMethod()
    destroy specificNFT
}
```

## Resource Safety

### Practice 16: Always Handle All Resources

**Every code path must explicitly handle resources.**

```cadence
// ✅ CORRECT: All paths handle resource
access(all) fun conditionalHandle(vault: @Vault, save: Bool) {
    if save {
        account.storage.save(<-vault, to: /storage/vault)
    } else {
        destroy vault
    }
    // Both paths handle the resource
}

// ❌ WRONG: Some paths don't handle resource
access(all) fun conditionalHandle(vault: @Vault, save: Bool) {
    if save {
        account.storage.save(<-vault, to: /storage/vault)
    }
    // COMPILE ERROR: vault not handled in else path
}
```

### Practice 17: Resource Cleanup in Error Cases

**Ensure resources are handled even when errors occur.**

```cadence
// ✅ CORRECT: Handle resource before panicking
access(all) fun riskyOperation(vault: @Vault) {
    if !someCondition() {
        destroy vault  // Handle resource first
        panic("Condition not met")
    }

    // Continue with vault
    account.storage.save(<-vault, to: /storage/vault)
}

// ❌ WRONG: Panic without handling resource
access(all) fun riskyOperation(vault: @Vault) {
    if !someCondition() {
        panic("Condition not met")  // vault is lost!
    }

    account.storage.save(<-vault, to: /storage/vault)
}
```

## Transaction Security

### Practice 18: Audit Transactions Like Contracts

**Transactions can contain arbitrary code and should be reviewed carefully.**

```cadence
// ⚠️ DANGEROUS: Transaction requests excessive permissions
transaction() {
    prepare(signer: auth(Storage, Contracts, Keys) &Account) {
        // What is this transaction doing?
        // Could deploy malicious contracts
        // Could modify keys
        // Could steal resources
    }
}

// ✅ SAFE: Transaction is transparent and minimal
transaction(amount: UFix64, to: Address) {
    let vault: @{FungibleToken.Vault}

    prepare(signer: auth(BorrowValue) &Account) {
        let vaultRef = signer.storage
            .borrow<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(from: /storage/vault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")

        self.vault <- vaultRef.withdraw(amount: amount)
    }

    execute {
        let receiver = getAccount(to)
            .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver)
            ?? panic("Could not borrow FungibleToken Receiver reference from /public/receiver")

        receiver.deposit(from: <-self.vault)
    }
}
```

### Practice 19: Understand Requested Entitlements

**Users should understand what they're authorizing.**

When signing transactions, verify:
- What entitlements are requested
- What resources are accessed
- What operations are performed
- Where resources are moved

```cadence
// Clear, auditable transaction
transaction(amount: UFix64) {
    // Explicitly states it needs BorrowValue
    prepare(signer: auth(BorrowValue) &Account) {
        // Clear: borrowing from specific path
        let vault = signer.storage.borrow<&Vault>(from: /storage/vault)
            ?? panic("Could not borrow Vault reference from /storage/vault")

        // Clear: withdrawing specific amount
        let withdrawn <- vault.withdraw(amount: amount)

        // Clear: depositing to specific location
        MyContract.deposit(vault: <-withdrawn)
    }
}
```

## Event Security

### Practice 20: Emit Events for Significant Actions

**Events provide audit trail and transparency.**

```cadence
access(all) event TokensWithdrawn(amount: UFix64, from: Address?)
access(all) event TokensDeposited(amount: UFix64, to: Address?)
access(all) event AdminActionPerformed(action: String, by: Address?)

access(all) resource Vault {
    access(self) var balance: UFix64

    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        self.balance = self.balance - amount

        // Emit event for transparency
        emit TokensWithdrawn(amount: amount, from: self.owner?.address)

        return <- create Vault(balance: amount)
    }

    access(all) fun deposit(from: @{FungibleToken.Vault}) {
        let amount = from.balance
        self.balance = self.balance + amount
        destroy from

        // Emit event for transparency
        emit TokensDeposited(amount: amount, to: self.owner?.address)
    }
}
```

## Testing and Verification

### Practice 21: Comprehensive Testing

**Test all security boundaries and edge cases.**

```cadence
import Test

// Test access control
access(all) fun testUnauthorizedAccess() {
    let vault <- createVault()

    // Should fail without proper entitlement
    Test.expectFailure(fun() {
        let ref = &vault as &Vault
        ref.withdraw(amount: 100.0)  // Should require entitlement
    })

    destroy vault
}

// Test capability validity
access(all) fun testInvalidCapability() {
    let invalidCap = getInvalidCapability()

    // Should handle gracefully
    if let ref = invalidCap.borrow() {
        Test.assert(false, message: "Should not borrow invalid capability")
    } else {
        // Correct behavior
        Test.assert(true)
    }
}

// Test resource handling
access(all) fun testResourceHandling() {
    let vault <- createVault()

    // All paths should handle resource
    if shouldKeep {
        account.storage.save(<-vault, to: /storage/test)
    } else {
        destroy vault
    }
}
```

## Security Checklist

When writing Cadence code, verify:

- [ ] All fields use `access(self)` or `access(contract)` by default
- [ ] Privileged operations use entitlements
- [ ] No capabilities in public fields, arrays, or dictionaries
- [ ] Capabilities validated before borrowing
- [ ] Minimal entitlements requested in transactions
- [ ] Resources handled in all code paths
- [ ] Types are as specific as possible
- [ ] User storage not trusted without validation
- [ ] Events emitted for significant actions
- [ ] Comprehensive tests for security boundaries

## Reference Links

- [Cadence Security Best Practices](https://cadence-lang.org/docs/security-best-practices)
- [Cadence Anti-Patterns](https://cadence-lang.org/docs/anti-patterns)
- [Access Control Documentation](https://cadence-lang.org/docs/language/access-control)
- [Capabilities Documentation](https://cadence-lang.org/docs/language/capabilities)
