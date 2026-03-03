# Cadence Capabilities and Security Rules

> **Purpose**: Comprehensive guidelines for creating, managing, and securing capabilities in Cadence. Capabilities are the foundation of Flow's capability-based security model.

## What Are Capabilities?

A capability represents **the right to access an object and perform certain operations on it**. Capabilities in Cadence are:

- **Unforgeable**: Cannot be created arbitrarily; must be issued by account owners
- **Transferable**: Can be passed to other accounts or stored
- **Revocable**: Can be deleted by the issuing account, invalidating all copies

## Capability Structure

```cadence
// Generic capability type
Capability<T: &Any>

// Properties
capability.address: Address    // Target account address
capability.id: UInt64          // Unique identifier per account

// Methods
capability.borrow(): T?        // Returns optional reference to target object
capability.check(): Bool       // Verifies capability targets compatible object
```

## Two Types of Capabilities

### 1. Storage Capabilities

Grant access to objects at specific storage paths:

```cadence
// Issue a storage capability
let vaultCap = account.capabilities.storage
    .issue<&FlowToken.Vault>(/storage/flowTokenVault)
```

### 2. Account Capabilities

Grant access to entire accounts:

```cadence
// Issue an account capability
let accountCap = account.capabilities.account
    .issue<auth(Storage, Keys) &Account>()
```

## Capability Lifecycle

### Phase 1: Issue

Create new capabilities using capability controllers:

```cadence
// Issue storage capability
let controller = account.capabilities.storage
    .issue<&MyResource>(/storage/myResource)

// Issue account capability
let accountController = account.capabilities.account
    .issue<&Account>()
```

**Important**: Issuing creates a controller that can be used to manage the capability later.

### Phase 2: Publish

Make capabilities publicly available:

```cadence
// Publish a capability at a public path
account.capabilities.publish(vaultCap, at: /public/flowTokenReceiver)
```

### Phase 3: Get/Borrow

Retrieve and use published capabilities:

```cadence
// Get a capability
let cap = getAccount(address)
    .capabilities.get<&{FungibleToken.Receiver}>(/public/flowTokenReceiver)

// Borrow the capability to get a reference
if let receiverRef = cap.borrow() {
    receiverRef.deposit(from: <-vault)
}

// Convenience: Get and borrow in one step
if let receiverRef = getAccount(address)
    .capabilities.borrow<&{FungibleToken.Receiver}>(/public/flowTokenReceiver) {
    receiverRef.deposit(from: <-vault)
}
```

### Phase 4: Revoke

Delete capability controllers to invalidate all copies:

```cadence
// Revoke a capability by deleting its controller
account.capabilities.storage.delete(controller.capabilityID)

// After deletion, all copies of this capability become invalid
```

## Secure Capability Patterns

### Pattern 1: Minimal Exposure with Interface Types

**Only publish a capability to an object if you want some of the functionality to be publicly accessible. If you do, only publish the concrete type with no entitlements.**

```cadence

access(all) resource Vault {
    access(self) var balance: UFix64

    access(all) view fun getBalance(): UFix64 {
        return self.balance
    }

    access(all) fun deposit(from: @{FungibleToken.Vault}) {
        // Implementation
    }

    // Private withdraw function NOT exposed through concrete type
    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        // Implementation
    }
}

// Issue and publish restricted capability
let vaultCap = account.capabilities.storage
    .issue<&Vault>(/storage/vault)  // Only Vault reference type
account.capabilities.publish(vaultCap, at: /public/vaultReceiver)

// ❌ WRONG: Exposing full resource with entitlements
let vaultCap = account.capabilities.storage
    .issue<auth(Withdraw) &Vault>(/storage/vault)  // Full access including withdraw!
account.capabilities.publish(vaultCap, at: /public/vaultReceiver)  // SECURITY FLAW
```

### Pattern 2: Entitled Capabilities for Privileged Access

Use entitlements to grant specific permissions:

```cadence
access(all) entitlement Withdraw
access(all) entitlement Admin

access(all) resource Vault {
    access(self) var balance: UFix64

    access(all) view fun getBalance(): UFix64 {
        return self.balance
    }

    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        // Only accessible with Withdraw entitlement
    }

    access(Admin) fun forceTransfer(to: Address, amount: UFix64) {
        // Only accessible with Admin entitlement
    }
}

// Create entitled capability (private - not published)
let withdrawCap = account.capabilities.storage
    .issue<auth(Withdraw) &Vault>(/storage/vault)

// Create admin capability (highly restricted)
let adminCap = account.capabilities.storage
    .issue<auth(Admin, Withdraw) &Vault>(/storage/vault)

// Public capability (no entitlements)
let publicCap = account.capabilities.storage
    .issue<&Vault>(/storage/vault)
account.capabilities.publish(publicCap, at: /public/vault)
```

### Pattern 3: Separate Public and Private Capabilities

```cadence
// Public capability - anyone can access
let publicCap = account.capabilities.storage
    .issue<&ConcreteTypeWithNoEntitlements>(/storage/resource)
account.capabilities.publish(publicCap, at: /public/resource)

// Private capability - stored for owner's use only
let privateCap = account.capabilities.storage
    .issue<auth(Admin) &Resource>(/storage/resource)
account.storage.save(privateCap, to: /storage/resourceAdmin)
```

## Capability Management Best Practices

### Practice 1: Use Capability Controllers

**Always retain controllers for capabilities you issue:**

```cadence
// ✅ CORRECT: Store controller for later management
let controller = account.capabilities.storage
    .issue<&MyResource>(/storage/myResource)

// Store controller for later use
account.storage.save(controller, to: /storage/myResourceController)

// Later: Revoke the capability
if let controller = account.storage.borrow<&StorageCapabilityController>
    (from: /storage/myResourceController) {
    account.capabilities.storage.delete(controller.capabilityID)
}

// ❌ WRONG: Losing reference to controller
let cap = account.capabilities.storage
    .issue<&MyResource>(/storage/myResource)
// No way to revoke this capability later!
```

### Practice 2: Tag Capabilities for Documentation

Use tags to document capability purposes:

```cadence
// Issue capability with descriptive tag
let controller = account.capabilities.storage
    .issue<&FlowToken.Vault>(/storage/flowTokenVault)

controller.setTag("FlowToken Receiver - Public Deposit Access")

// Later: Audit capabilities by tags
let controllers = account.capabilities.storage.getControllers(forPath: /storage/flowTokenVault)
for controller in controllers {
    log(controller.tag)
}
```

### Practice 3: Regular Capability Audits

Periodically review and revoke unused capabilities:

```cadence
// Audit script example
access(all) fun auditCapabilities(account: &Account) {
    // Get all storage paths
    let paths = account.storage.storagePaths

    for path in paths {
        let controllers = account.capabilities.storage.getControllers(forPath: path)

        for controller in controllers {
            log("Path: ".concat(path.toString()))
            log("Cap ID: ".concat(controller.capabilityID.toString()))
            log("Type: ".concat(controller.borrowType.identifier))
            log("Tag: ".concat(controller.tag ?? "No tag"))
        }
    }
}
```

### Practice 4: Principle of Least Privilege

**Issue capabilities with minimum necessary entitlements:**

```cadence
// ❌ AVOID: Over-privileged capability
let cap = account.capabilities.storage
    .issue<auth(Admin, Withdraw, Mutate, Insert, Remove) &Resource>(/storage/resource)

// ✅ CORRECT: Minimal entitlements for specific use case
let cap = account.capabilities.storage
    .issue<auth(Withdraw) &Resource>(/storage/resource)
```

## Capability Checking Patterns

### Check Before Borrow

Always verify capability validity before borrowing:

```cadence
// ✅ CORRECT: Check then borrow
let cap = getAccount(address)
    .capabilities.get<&{FungibleToken.Receiver}>(/public/receiver)

if cap.check() {
    if let receiver = cap.borrow() {
        receiver.deposit(from: <-vault)
    }
} else {
    panic("Could not borrow FungibleToken Receiver reference from /public/receiver")
}

// ✅ ALSO CORRECT: Borrow returns optional
let cap = getAccount(address)
    .capabilities.get<&{FungibleToken.Receiver}>(/public/receiver)

if let receiver = cap.borrow() {
    receiver.deposit(from: <-vault)
} else {
    panic("Could not borrow FungibleToken Receiver reference from /public/receiver")
}

// ❌ WRONG: Force unwrap without checking
let receiver = cap.borrow()!  // May panic!
receiver.deposit(from: <-vault)
```

### Defensive Borrowing Pattern

```cadence
access(all) fun safeDeposit(
    receiverAddress: Address,
    vault: @{FungibleToken.Vault}
): Bool {
    let cap = getAccount(receiverAddress)
        .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver)

    if let receiver = cap {
        receiver.deposit(from: <-vault)
        return true
    } else {
        // Handle failure - don't lose resource
        destroy vault
        return false
    }
}
```

## Capability Retargeting

Change storage paths for existing capability controllers:

```cadence
// Create capability
let controller = account.capabilities.storage
    .issue<&MyResource>(/storage/oldPath)

// Later: Move resource to new path
let resource <- account.storage.load<@MyResource>(from: /storage/oldPath)
account.storage.save(<-resource, to: /storage/newPath)

// Retarget capability to new path
controller.retarget(/storage/newPath)

// Capability now points to new location
```

**Note**: Retargeting only works for storage capabilities, not account capabilities.

## Common Security Vulnerabilities

### Vulnerability 1: Publishing Full Resource Access

```cadence
// ❌ CRITICAL VULNERABILITY
access(all) resource Vault {
    access(all) var balance: UFix64

    access(all) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        // Anyone can withdraw!
    }
}

// Publishing full access
let vaultCap = account.capabilities.storage
    .issue<&Vault>(/storage/vault)
account.capabilities.publish(vaultCap, at: /public/vault)

// ✅ FIX: Use interface and entitlements
access(all) entitlement Withdraw

access(all) resource Vault {
    access(self) var balance: UFix64

    access(all) view fun getBalance(): UFix64 {
        return self.balance
    }

    access(Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
        // Requires Withdraw entitlement
    }
}

// Publish concrete type only with no entitlements
let publicCap = account.capabilities.storage
    .issue<&Vault>(/storage/vault)
account.capabilities.publish(publicCap, at: /public/vault)
```

### Vulnerability 2: Forgetting to Revoke Capabilities

```cadence
// ❌ PROBLEM: Issued capability but lost control
let cap = account.capabilities.storage
    .issue<auth(Admin) &AdminResource>(/storage/admin)

// Gave capability to employee
employeeAccount.save(cap, to: /storage/adminAccess)

// Employee leaves company - capability still valid!

// ✅ FIX: Store controller and revoke when needed
let controller = account.capabilities.storage
    .issue<auth(Admin) &AdminResource>(/storage/admin)

// Store controller
account.storage.save(controller, to: /storage/adminController)

// Give capability to employee
let cap = controller.capability
employeeAccount.save(cap, to: /storage/adminAccess)

// Employee leaves - revoke capability
if let ctrl = account.storage.borrow<&StorageCapabilityController>
    (from: /storage/adminController) {
    account.capabilities.storage.delete(ctrl.capabilityID)
}
// Now employee's capability is invalid
```

### Vulnerability 3: Type Confusion

```cadence
// ❌ PROBLEM: Wrong type annotation
let cap = getAccount(address)
    .capabilities.get<&{WrongInterface}>(/public/resource)

if let ref = cap.borrow() {
    // This will never execute if types don't match
    ref.doSomething()
}

// ✅ FIX: Use correct type
let cap = getAccount(address)
    .capabilities.get<&{CorrectInterface}>(/public/resource)

if let ref = cap.borrow() {
    ref.doSomething()
}
```

## AI Agent Generation Guidelines

### When Issuing Capabilities

1. **Default to concrete type**
2. **Ensure that concrete type has entitlements for any privileged functionality**
2. **Use entitlements for privileged access**
3. **Store capability controllers for management**
4. **Add descriptive tags**
5. **Only publish when necessary**

```cadence
// AI should generate this pattern:

access(all) resource MyResource {
    access(self) var data: String

    access(all) view fun publicMethod(): String {
        return self.data
    }

    access(Admin) fun privilegedMethod() {
        // Requires entitlement
    }
}

// Step 2: Issue restrictive public capability
let publicController = account.capabilities.storage
    .issue<&MyResource>(/storage/myResource)
publicController.setTag("Public read-only access")

let publicCap = publicController.capability
account.capabilities.publish(publicCap, at: /public/myResource)

// Step 3: Issue entitled private capability
let adminController = account.capabilities.storage
    .issue<auth(Admin) &MyResource>(/storage/myResource)
adminController.setTag("Admin privileged access")

// Step 4: Store controllers for management
account.storage.save(publicController, to: /storage/myResourcePublicController)
account.storage.save(adminController, to: /storage/myResourceAdminController)
```

### Capability Usage Checklist

When generating code that uses capabilities:

- [ ] Capability type matches expected interface
- [ ] `check()` or optional unwrap before borrowing
- [ ] Defensive handling of borrow failures
- [ ] No force unwrapping (`!`) of borrowed references
- [ ] Resource cleanup in case of borrow failure

## Reference Patterns

### Pattern: Safe Resource Transfer via Capability

```cadence
access(all) fun transferResource(
    to: Address,
    resource: @MyResource
) {
    let receiverCap = getAccount(to)
        .capabilities.borrow<&{ResourceReceiver}>(/public/receiver)

    if let receiver = receiverCap {
        receiver.deposit(resource: <-resource)
    } else {
        // Handle failure - don't lose resource
        log("Warning: Could not transfer resource to \(to)")

        // Return to sender or destroy based on use case
        destroy resource
    }
}
```

### Pattern: Capability-Based Access Control

```cadence
access(all) resource ProtectedResource {
    access(self) var data: String

    // Verify caller has specific capability
    access(all) fun updateData(
        newData: String,
        proof: Capability<auth(Update) &ProtectedResource>
    ) {
        // Verify proof capability is valid and matches this resource
        if proof.check() && proof.address == self.owner?.address {
            if let authorizedRef = proof.borrow() {
                self.data = newData
            }
        } else {
            panic("Authorization proof capability is invalid or does not match this resource's owner")
        }
    }
}
```

## Advanced: Entitlement Mappings with Capabilities

Use entitlement mappings to automatically propagate entitlements through nested objects:

```cadence
entitlement mapping MyMap {
    AdminAccess -> Admin
    UserAccess -> User
}

access(all) resource Container {
    access(self) let innerResource: @InnerResource

    access(mapping MyMap) fun getInner(): auth(mapping MyMap) &InnerResource {
        return &self.innerResource as auth(mapping MyMap) &InnerResource
    }
}

// Capability with AdminAccess automatically grants Admin on inner resource
let containerCap = account.capabilities.storage
    .issue<auth(AdminAccess) &Container>(/storage/container)

if let container = containerCap.borrow() {
    let inner = container.getInner()  // Automatically has Admin entitlement
    inner.adminFunction()  // Works because of mapping
}
```

## Testing Capabilities

```cadence
import Test

access(all) fun testCapabilityLifecycle() {
    let account = Test.createAccount()

    // Issue capability
    let controller = account.capabilities.storage
        .issue<&MyResource>(/storage/myResource)

    let cap = controller.capability

    // Test: Capability should be valid
    Test.expect(cap.check(), Test.equal(true))

    // Test: Should be able to borrow
    Test.expect(cap.borrow(), Test.not(Test.equal(nil)))

    // Revoke capability
    account.capabilities.storage.delete(controller.capabilityID)

    // Test: Capability should now be invalid
    Test.expect(cap.check(), Test.equal(false))
    Test.expect(cap.borrow(), Test.equal(nil))
}
```

## Reference Links

- [Cadence Capabilities Documentation](https://cadence-lang.org/docs/language/capabilities)
- [Cadence Access Control Documentation](https://cadence-lang.org/docs/language/access-control)
- [Flow Capability-Based Security Model](https://flow.com/post/flow-blockchain-cadence-programming-language-resources-assets)
