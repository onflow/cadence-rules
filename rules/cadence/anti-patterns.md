# Cadence Anti-Patterns

> **Purpose**: Identify and prevent common anti-patterns that create security vulnerabilities or maintainability issues in Cadence code. These patterns should be actively avoided in all Cadence development.

## Core Principle: Security Through Restriction

Anti-patterns typically arise from **granting too much access** or **assuming too much trust**. The secure approach is to minimize permissions and validate all external interactions.

## Anti-Pattern 1: Fully Authorized Account References as Parameters

### The Problem

**Passing `auth(...) &Account` to functions grants dangerous privileges.**

When a function receives an authorized account reference, it has direct access to sensitive operations like storage writes, contract deployment, and key management. This creates a vector for malicious contract upgrades.

```cadence
// ❌ CRITICAL ANTI-PATTERN: Authorized account parameter
access(all) contract DangerousContract {
    // Accepts fully authorized account
    access(all) fun processAccount(account: auth(Storage) &Account) {
        // This function can do ANYTHING with the account!
        // Could steal all resources
        // Could deploy malicious contracts
        // Could modify any storage

        // Even if current implementation is safe,
        // a malicious upgrade could drain accounts
    }
}

// Usage in transaction - very dangerous!
transaction() {
    prepare(signer: auth(Storage) &Account) {
        // Passing full account access to contract
        DangerousContract.processAccount(account: signer)
    }
}
```

### Why It's Dangerous

1. **Malicious upgrades**: Contract owner could upgrade to steal resources
2. **Unintended access**: Function can access unrelated storage paths
3. **No revocation**: Once called, can't undo account access
4. **Hidden risk**: Users may not understand transaction implications

### The Fix

**Use resources, capabilities, or keep storage operations in transactions.**

```cadence
// ✅ CORRECT: Use capabilities for authentication
access(all) contract SecureContract {
    // Accept specific capability instead of account
    access(all) fun processVault(
        vaultCap: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>
    ) {
        // Can only access the specific capability
        // Limited, revocable access
        if let vault = vaultCap.borrow() {
            // Safe operations on vault only
        }
    }
}

// ✅ BETTER: Perform operations in transaction
transaction() {
    prepare(signer: auth(Storage) &Account) {
        // All storage access happens here, in user-controlled code
        let vault = signer.storage
            .borrow<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(from: /storage/vault)
            ?? panic("Could not borrow FungibleToken Vault reference from /storage/vault")

        // Pass only the reference, not account
        SecureContract.processVault(vault: vault)
    }
}

// ✅ BEST: Use resource-based authentication
access(all) resource AdminBadge {
    access(all) let id: UInt64
    init(id: UInt64) { self.id = id }
}

access(all) contract SecureContract {
    // Verify ownership of resource
    access(all) fun adminFunction(badge: &AdminBadge) {
        // Badge proves authorization
        // No account access needed
    }
}
```

### Exception: Transaction Prepare Blocks

The **only** acceptable place for authorized account references is in transaction `prepare` blocks, where users explicitly authorize operations.

```cadence
// ✅ ACCEPTABLE: Transaction prepare block
transaction() {
    prepare(signer: auth(Storage, Keys) &Account) {
        // User explicitly authorizes this
        // Contained within transaction they sign
        // Not in upgradeable contract
    }
}
```

## Anti-Pattern 2: Public Functions and Fields

### The Problem

**Unintentionally using `access(all)` exposes internal implementation.**

```cadence
// ❌ ANTI-PATTERN: Everything public
access(all) contract BadContract {
    // Exposed internal counter
    access(all) var internalCounter: UInt64

    // Exposed admin address
    access(all) let adminAddress: Address

    // Exposed helper function
    access(all) fun internalHelper(): String {
        return "internal data"
    }

    // Exposed admin function - CRITICAL!
    access(all) fun resetSystem() {
        self.internalCounter = 0
        // Anyone can call this!
    }

    init() {
        self.internalCounter = 0
        self.adminAddress = self.account.address
    }
}
```

### Why It's Dangerous

1. **No access control**: Anyone can call functions or read fields
2. **Exposed internals**: Implementation details become API contract
3. **Security vulnerabilities**: Admin operations accessible to all
4. **Cannot change**: Public API is permanent, hard to refactor

### The Fix

**Default to private, explicitly make public only what's necessary.**

```cadence
// ✅ CORRECT: Restrictive access control
access(all) contract GoodContract {
    // Private internal state
    access(self) var internalCounter: UInt64
    access(self) let adminAddress: Address

    // Private helper
    access(self) fun internalHelper(): String {
        return "internal data"
    }

    // Public view function (safe - read-only)
    access(all) view fun getCounter(): UInt64 {
        return self.internalCounter
    }

    // Entitled admin function
    access(Admin) fun resetSystem() {
        self.internalCounter = 0
    }

    // Public but safe operation, though typically state changing operations
    // require some sort of authentication
    access(all) fun incrementCounter() {
        self.internalCounter = self.internalCounter + 1
    }

    init() {
        self.internalCounter = 0
        self.adminAddress = self.account.address
    }
}
```

### Special Case: Complex Fields

**Never make complex fields public, even as constants.**

```cadence
// ❌ WRONG: Public array/dictionary/resource/capability
access(all) contract BadContract {
    access(all) let adminCapabilities: [Capability<&Admin>]
    access(all) var userData: {Address: UserData}
    access(all) var vaults: @{UInt64: Vault}

    init() {
        self.adminCapabilities = []
        self.userData = {}
        self.vaults <- {}
    }

    destroy() {
        destroy self.vaults
    }
}

// ✅ CORRECT: Private complex fields
access(all) contract GoodContract {
    access(self) let adminCapabilities: [Capability<&Admin>]
    access(self) var userData: {Address: UserData}
    access(self) var vaults: @{UInt64: Vault}

    // Expose through controlled functions
    access(Admin) fun getAdminCapability(index: Int): Capability<&Admin>? {
        if index < self.adminCapabilities.length {
            return self.adminCapabilities[index]
        }
        return nil
    }

    access(all) view fun getUserData(address: Address): UserData? {
        return self.userData[address]
    }

    init() {
        self.adminCapabilities = []
        self.userData = {}
        self.vaults <- {}
    }

    destroy() {
        destroy self.vaults
    }
}
```

## Anti-Pattern 3: Capability-Typed Public Fields

### The Problem

**Public capability fields can be copied by anyone.**

Capabilities are value types, so storing them in public fields allows anyone to copy and use them.

```cadence
// ❌ CRITICAL ANTI-PATTERN: Public capability field
access(all) contract VulnerableContract {
    // Anyone can copy this capability!
    access(all) let adminCapability: Capability<auth(Admin) &Admin>

    // Public array of capabilities - even worse!
    access(all) let minterCapabilities: [Capability<&Minter>]

    init(adminCap: Capability<auth(Admin) &Admin>) {
        self.adminCapability = adminCap
        self.minterCapabilities = []
    }
}

// Attacker code:
let stolenAdminCap = VulnerableContract.adminCapability
let stolenMinterCap = VulnerableContract.minterCapabilities[0]
// Attacker now has admin/minter access!
```

### Why It's Dangerous

1. **Unrestricted copying**: Any account can duplicate the capability
2. **Cannot revoke**: Copies remain valid even if original is revoked
3. **Privilege escalation**: Users gain unauthorized access
4. **Permanent exposure**: Cannot fix without contract upgrade

### The Fix

**Store capabilities privately, expose functionality through controlled interfaces.**

```cadence
// ✅ CORRECT: Private capability storage
access(all) contract SecureContract {
    // Private capability field
    access(self) let adminCapability: Capability<auth(Admin) &Admin>
    access(self) var minterCapabilities: [Capability<&Minter>]

    init(adminCap: Capability<auth(Admin) &Admin>) {
        self.adminCapability = adminCap
        self.minterCapabilities = []
    }

    // Expose functionality, not capability
    access(all) fun performAdminAction() {
        if let admin = self.adminCapability.borrow() {
            admin.doSomething()
        }
    }

    // Or use entitlements to control access
    access(Admin) fun executeAdminFunction() {
        if let admin = self.adminCapability.borrow() {
            admin.execute()
        }
    }
}
```

### Alternative: Explicit Public Capabilities

If capabilities must be public, place them explicitly in account public storage:

```cadence
// ✅ ACCEPTABLE: Explicit public capability
transaction() {
    prepare(signer: auth(IssueStorageCapabilityController, PublishCapability) &Account) {
        // Create capability
        let controller = signer.capabilities.storage
            .issue<&Resource>(/storage/resource)

        // Explicitly publish to public path
        signer.capabilities.publish(controller.capability, at: /public/resource)
        // Clear intent: this is meant to be public
    }
}
```

## Anti-Pattern 4: Public Admin Resource Creation

### The Problem

**Public functions that create admin resources allow anyone to become admin.**

```cadence
// ❌ CRITICAL ANTI-PATTERN: Public admin creation
access(all) contract VulnerableContract {
    access(all) resource Admin {
        access(all) fun deleteAllData() {
            // Admin operations
        }
    }

    // Anyone can call this!
    access(all) fun createAdmin(): @Admin {
        return <- create Admin()
    }

    // Or this variant:
    access(all) fun setupAdmin(account: auth(SaveValue) &Account) {
        let admin <- create Admin()
        account.storage.save(<-admin, to: /storage/admin)
        // Anyone can become admin!
    }
}

// Attacker code:
let admin <- VulnerableContract.createAdmin()
// Attacker is now admin!
```

### Why It's Dangerous

1. **Unrestricted access**: Anyone can create admin resources
2. **No authentication**: No verification of authorization
3. **Privilege escalation**: Regular users become admins
4. **Cannot revoke**: Admin resources cannot be taken back

### The Fix

**Create admin resource in contract initializer, restrict creation to admin-only.**

```cadence
// ✅ CORRECT: Admin created in initializer only
access(all) contract SecureContract {
    access(all) resource Admin {
        // Admin can create more admins if needed
        access(all) fun createAdmin(): @Admin {
            return <- create Admin()
        }

        access(all) fun performAdminAction() {
            // Admin operations
        }
    }

    init() {
        // Create single admin instance
        let admin <- create Admin()

        // Save to deployer's account
        self.account.storage.save(<-admin, to: /storage/admin)

        // No public creation function!
    }
}

// ✅ ALSO CORRECT: Admin-only creation
access(all) contract SecureContract {
    access(all) resource Admin {
        // Only existing admins can create new admins
        access(all) fun createAdmin(): @Admin {
            return <- create Admin()
        }
    }

    // No public creation function
    // Must have existing admin to create new ones

    init() {
        let admin <- create Admin()
        self.account.storage.save(<-admin, to: /storage/admin)
    }
}
```

### Pattern: Delegation Through References

If multiple accounts need admin access:

```cadence
// ✅ CORRECT: Capability-based delegation
access(all) contract SecureContract {
    access(all) resource Admin {
        access(all) fun performAction() {
            // Admin operation
        }
    }

    init(adminAccount: auth(SaveValue, IssueStorageCapabilityController) &Account) {
        // Create single admin
        let admin <- create Admin()
        adminAccount.storage.save(<-admin, to: /storage/admin)

        // Create capability for delegation
        let adminCap = adminAccount.capabilities.storage
            .issue<&Admin>(/storage/admin)

        // Store capability in contract
        self.account.storage.save(adminCap, to: /storage/adminCap)
    }

    // Delegate admin access through stored capability
    access(all) fun performAdminAction() {
        let adminCap = self.account.storage
            .borrow<Capability<&Admin>>(from: /storage/adminCap)!

        if let admin = adminCap.borrow() {
            admin.performAction()
        }
    }
}
```

## Anti-Pattern 5: State Modification in Public Struct Initializers

### The Problem

**Structs must be public to be usable, but anyone can create them.**

If struct initializers modify contract state, attackers can repeatedly instantiate structs to manipulate state.

```cadence
// ❌ CRITICAL ANTI-PATTERN: State modification in struct init
access(all) contract VulnerableContract {
    access(self) var totalMinted: UInt64
    access(self) var nextID: UInt64

    // Public struct (must be public to use)
    access(all) struct NFTData {
        access(all) let id: UInt64
        access(all) let serialNumber: UInt64

        init() {
            // PROBLEM: Modifies contract state!
            VulnerableContract.totalMinted = VulnerableContract.totalMinted + 1
            VulnerableContract.nextID = VulnerableContract.nextID + 1

            self.id = VulnerableContract.nextID
            self.serialNumber = VulnerableContract.totalMinted
        }
    }

    init() {
        self.totalMinted = 0
        self.nextID = 0
    }
}

// Attacker code:
// Create unlimited NFTData instances, exhausting IDs!
let fake1 = VulnerableContract.NFTData()
let fake2 = VulnerableContract.NFTData()
let fake3 = VulnerableContract.NFTData()
// Contract state corrupted!
```

### Real-World Example: NBA Top Shot Bug

This exact pattern existed in NBA Top Shot, allowing anyone to exhaust the ID sequence without actually creating valid NFTs.

### Why It's Dangerous

1. **Uncontrolled instantiation**: Anyone can create structs
2. **State corruption**: Contract state manipulated without authorization
3. **Resource exhaustion**: ID sequences or counters exhausted
4. **No rate limiting**: Can be called unlimited times

### The Fix

**Move state modifications to protected resource functions.**

```cadence
// ✅ CORRECT: State modification in protected resource
access(all) contract SecureContract {
    access(self) var totalMinted: UInt64
    access(self) var nextID: UInt64

    // Public struct (no state modification)
    access(all) struct NFTData {
        access(all) let id: UInt64
        access(all) let serialNumber: UInt64

        // Initialize with provided values only
        init(id: UInt64, serialNumber: UInt64) {
            self.id = id
            self.serialNumber = serialNumber
        }
    }

    // Protected resource for minting
    access(all) resource Minter {
        // Only minter can create NFTData with state modification
        access(all) fun mint(): NFTData {
            // State modification protected by resource ownership
            SecureContract.totalMinted = SecureContract.totalMinted + 1
            SecureContract.nextID = SecureContract.nextID + 1

            return NFTData(
                id: SecureContract.nextID,
                serialNumber: SecureContract.totalMinted
            )
        }
    }

    init() {
        self.totalMinted = 0
        self.nextID = 0

        // Create single minter
        let minter <- create Minter()
        self.account.storage.save(<-minter, to: /storage/minter)
    }
}
```

### Alternative: Event Emission

Similarly, avoid emitting events from struct initializers:

```cadence
// ❌ WRONG: Event emission in struct init
access(all) contract VulnerableContract {
    access(all) event ItemCreated(id: UInt64)

    access(all) struct Item {
        access(all) let id: UInt64

        init(id: UInt64) {
            self.id = id
            emit ItemCreated(id: id)  // Uncontrolled event emission!
        }
    }
}

// ✅ CORRECT: Event emission in protected function
access(all) contract SecureContract {
    access(all) event ItemCreated(id: UInt64)

    access(all) struct Item {
        access(all) let id: UInt64

        init(id: UInt64) {
            self.id = id
            // No event emission
        }
    }

    access(all) resource Creator {
        access(all) fun createItem(id: UInt64): Item {
            let item = Item(id: id)
            emit ItemCreated(id: id)  // Protected by resource
            return item
        }
    }
}
```

## Anti-Pattern Detection Checklist

When reviewing Cadence code, check for:

- [ ] Functions accepting `auth(...) &Account` parameters
- [ ] Public fields (especially complex types like arrays, dictionaries, resources, capabilities)
- [ ] Capability-typed public fields
- [ ] Public functions that create admin/privileged resources
- [ ] State modifications in struct initializers
- [ ] Event emissions in struct initializers
- [ ] Over-use of `access(all)` modifier
- [ ] Missing access control on sensitive operations

## Migration Guide

### Fixing Existing Anti-Patterns

1. **Identify**: Search codebase for anti-patterns
2. **Assess risk**: Determine if exploitable
3. **Plan fix**: Design secure alternative
4. **Implement**: Create new contract or upgrade
5. **Migrate**: Move data and functionality
6. **Verify**: Test security boundaries

### Example Migration

```cadence
// Old contract with anti-patterns
access(all) contract OldContract {
    access(all) var counter: UInt64
    access(all) let adminCap: Capability<&Admin>

    access(all) fun createAdmin(): @Admin {
        return <- create Admin()
    }
}

// New secure contract
access(all) contract NewContract {
    access(self) var counter: UInt64
    access(self) let adminCap: Capability<&Admin>

    access(all) view fun getCounter(): UInt64 {
        return self.counter
    }

    // Admin creation removed - use init() only
}
```

## Reference Links

- [Cadence Anti-Patterns Documentation](https://cadence-lang.org/docs/anti-patterns)
- [Cadence Security Best Practices](https://cadence-lang.org/docs/security-best-practices)
- [NBA Top Shot Bug Post-Mortem](https://flow.com/post/flow-blockchain-cadence-programming-language-update)
