# Cadence Reference Rules

> **Purpose**: Define comprehensive guidelines for working with references in Cadence. References provide non-owning access to values without transferring ownership.

## Core Philosophy: References as Views

References are **non-owning pointers** that provide temporary access to values. They:
- Don't transfer ownership
- Can't be stored (ephemeral)
- Use entitlements for authorization
- Allow read/write access without moving resources

## Creating References

### Basic Reference Creation

Use the `&` operator to create references:

```cadence
let resource <- createResource()

// Create reference to resource
let resourceRef = &resource as &MyResource

// Use reference (resource still owned by original variable)
resourceRef.someFunction()

// Original resource still valid
destroy resource
```

### Reference Type Annotation

References require explicit types:

```cadence
// With type annotation
let ref: &MyResource = &resource as &MyResource

// Reference to interface
let interfaceRef: &{MyInterface} = &resource as &{MyInterface}

// Reference to intersection type
let multiRef: &{InterfaceA, InterfaceB} = &resource as &{InterfaceA, InterfaceB}
```

## Authorized References

### Creating Authorized References

Use `auth()` to create references with entitlements:

```cadence
access(all) entitlement Owner
access(all) entitlement Admin

access(all) resource Vault {
    access(self) var balance: UFix64

    // Requires Owner entitlement
    access(Owner) fun withdraw(amount: UFix64): @Vault {
        self.balance = self.balance - amount
        return <- create Vault(balance: amount)
    }

    // Requires Admin entitlement
    access(Admin) fun setBalance(newBalance: UFix64) {
        self.balance = newBalance
    }

    init(balance: UFix64) {
        self.balance = balance
    }
}

// Create authorized reference
let vault <- create Vault(balance: 100.0)

// Reference with Owner entitlement
let ownerRef = &vault as auth(Owner) &Vault
ownerRef.withdraw(amount: 10.0)  // Allowed

// Reference with Admin entitlement
let adminRef = &vault as auth(Admin) &Vault
adminRef.setBalance(newBalance: 200.0)  // Allowed

// Reference with both entitlements
let fullRef = &vault as auth(Owner, Admin) &Vault
fullRef.withdraw(amount: 10.0)  // Allowed
fullRef.setBalance(newBalance: 150.0)  // Allowed

destroy vault
```

### Entitlement Combinations

```cadence
// Single entitlement
let ref1 = &resource as auth(Owner) &Resource

// Multiple entitlements (requires ALL)
let ref2 = &resource as auth(Owner, Admin) &Resource

// Disjunction (requires ANY) - use | in entitlement definition
access(all) entitlement mapping MyMap {
    Owner -> OwnerAccess
    Admin -> AdminAccess
}
```

## Reference to Storage

### Borrowing from Storage

Most common use case - borrowing from account storage:

```cadence
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Borrow reference from storage
        let vaultRef = signer.storage
            .borrow<&Vault>(from: /storage/vault)
            ?? panic("Vault not found")

        // Use reference
        log(vaultRef.balance)

        // Resource remains in storage
    }
}
```

### Authorized Storage References

```cadence
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Borrow with entitlements
        let vaultRef = signer.storage
            .borrow<auth(FungibleToken.Withdraw) &Vault>(from: /storage/vault)
            ?? panic("Vault not found")

        // Can call entitled functions
        let withdrawn <- vaultRef.withdraw(amount: 10.0)
        destroy withdrawn
    }
}
```

## Reference Validity

### Valid References

References remain valid as long as the referenced value exists:

```cadence
// ✅ CORRECT: Reference valid throughout scope
let resource <- createResource()
let ref = &resource as &Resource

ref.someFunction()  // Valid
ref.anotherFunction()  // Valid

destroy resource
// ref is now invalid (but scope ended)
```

### Invalid References

**Resource references become invalid when resource is moved or destroyed:**

```cadence
// ❌ WRONG: Using reference after move
let resource <- createResource()
let ref = &resource as &Resource

let moved <- resource  // resource moved
ref.someFunction()  // RUNTIME ERROR: resource moved, ref invalid

destroy moved
```

```cadence
// ❌ WRONG: Using reference after destroy
let resource <- createResource()
let ref = &resource as &Resource

destroy resource  // resource destroyed
ref.someFunction()  // RUNTIME ERROR: resource destroyed, ref invalid
```

### Safe Reference Usage Pattern

```cadence
// ✅ CORRECT: Use reference before moving/destroying
let resource <- createResource()
let ref = &resource as &Resource

// Use reference
let value = ref.getValue()
ref.someFunction()

// Now move/destroy
destroy resource
```

## References Cannot Be Stored

### The Ephemeral Rule

References are **ephemeral** - they cannot be saved to storage:

```cadence
// ❌ COMPILE ERROR: Cannot store references
access(all) resource Container {
    access(self) let vaultRef: &Vault  // COMPILE ERROR

    init(vaultRef: &Vault) {
        self.vaultRef = vaultRef  // Cannot store reference
    }
}
```

### Use Capabilities for Persistence

Instead of storing references, store capabilities:

```cadence
// ✅ CORRECT: Store capability, not reference
access(all) resource Container {
    access(self) let vaultCap: Capability<&Vault>

    init(vaultCap: Capability<&Vault>) {
        self.vaultCap = vaultCap  // Can store capabilities
    }

    access(all) fun useVault() {
        // Borrow reference when needed
        if let vaultRef = self.vaultCap.borrow() {
            vaultRef.someFunction()
        }
    }
}
```

## Reference Type Relationships

### Covariance

References are covariant - `&T` is a subtype of `&U` when `T` is a subtype of `U`:

```cadence
access(all) resource interface Animal {
    access(all) fun makeSound(): String
}

access(all) resource Dog: Animal {
    access(all) fun makeSound(): String {
        return "Woof"
    }

    access(all) fun wagTail() {
        // Dog-specific function
    }
}

let dog <- create Dog()

// Dog reference is a subtype of Animal reference
let animalRef: &{Animal} = &dog as &{Animal}  // Valid
animalRef.makeSound()  // Can call Animal interface functions

// But cannot call Dog-specific functions through Animal reference
// animalRef.wagTail()  // COMPILE ERROR

destroy dog
```

## Dereferencing

### Primitive Dereferencing

Only primitive values can be dereferenced using `*`:

```cadence
let num: Int = 42
let numRef = &num as &Int

// Dereference to get copy
let numCopy = *numRef  // Creates independent copy

log(num)      // 42
log(numCopy)  // 42
```

### Non-Primitive Types Cannot Be Dereferenced

```cadence
// ❌ CANNOT dereference composite types
let resource <- createResource()
let ref = &resource as &Resource

// let copy = *ref  // COMPILE ERROR: Cannot dereference resources

// ❌ CANNOT dereference structs
let myStruct = MyStruct()
let structRef = &myStruct as &MyStruct

// let structCopy = *structRef  // COMPILE ERROR

destroy resource
```

## Reference Patterns

### Pattern 1: Safe Borrowing

```cadence
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Borrow, use, done - reference doesn't outlive scope
        if let vaultRef = signer.storage.borrow<&Vault>(from: /storage/vault) {
            log(vaultRef.balance)
            // Reference automatically invalid after scope
        }
    }
}
```

### Pattern 2: Authorized Operations

```cadence
access(all) entitlement Withdraw

access(all) resource Vault {
    access(self) var balance: UFix64

    access(all) view fun getBalance(): UFix64 {
        return self.balance
    }

    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
        self.balance = self.balance - amount
        return <- create Vault(balance: amount)
    }

    init(balance: UFix64) {
        self.balance = balance
    }
}

transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Unauthorized reference - read-only
        let publicRef = signer.storage.borrow<&Vault>(from: /storage/vault)
        publicRef?.getBalance()  // OK
        // publicRef?.withdraw(amount: 10.0)  // COMPILE ERROR: needs Withdraw entitlement

        // Authorized reference - can withdraw
        let authorizedRef = signer.storage
            .borrow<auth(Withdraw) &Vault>(from: /storage/vault)
        let withdrawn <- authorizedRef?.withdraw(amount: 10.0)  // OK
        destroy withdrawn
    }
}
```

### Pattern 3: Interface References

```cadence
access(all) resource interface VaultPublic {
    access(all) fun getBalance(): UFix64
    access(all) fun deposit(from: @{FungibleToken.Vault})
}

access(all) resource Vault: VaultPublic {
    access(self) var balance: UFix64

    access(all) fun getBalance(): UFix64 {
        return self.balance
    }

    access(all) fun deposit(from: @{FungibleToken.Vault}) {
        // Implementation
    }

    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
        // Privileged operation
    }

    init(balance: UFix64) {
        self.balance = balance
    }
}

// Reference to interface hides privileged operations
transaction() {
    execute {
        let vaultRef = getAccount(address)
            .capabilities.borrow<&{VaultPublic}>(/public/vault)
            ?? panic("Vault not found")

        vaultRef.getBalance()  // OK
        vaultRef.deposit(from: <-vault)  // OK
        // vaultRef.withdraw(amount: 10.0)  // COMPILE ERROR: not in interface
    }
}
```

### Pattern 4: Capability Borrowing

```cadence
// Most common pattern - borrow from capability
let cap = getAccount(address)
    .capabilities.get<&{VaultPublic}>(/public/vault)

if let vaultRef = cap.borrow() {
    // Use reference within scope
    log(vaultRef.getBalance())
}
// Reference invalid after scope
```

## Reference Safety Rules

### Rule 1: Don't Store References

**References cannot be stored, only used within scope.**

```cadence
// ❌ WRONG: Trying to store reference
access(self) let storedRef: &Vault  // COMPILE ERROR

// ✅ CORRECT: Store capability
access(self) let vaultCap: Capability<&Vault>
```

### Rule 2: References Don't Transfer Ownership

**Creating a reference doesn't move the value.**

```cadence
let resource <- createResource()
let ref = &resource as &Resource

// Both valid
ref.someFunction()
destroy resource  // Still own the resource
```

### Rule 3: Check Reference Validity

**For resource references, ensure resource hasn't been moved/destroyed.**

```cadence
// ✅ CORRECT: Use reference before moving
let resource <- createResource()
let ref = &resource as &Resource

let value = ref.getValue()  // Use while valid

let moved <- resource  // Now ref becomes invalid
// Don't use ref after this point
```

### Rule 4: Use Appropriate Entitlements

**Grant minimal entitlements needed.**

```cadence
// ✅ GOOD: Minimal entitlement
let readRef = &resource as &Resource  // No entitlements - read-only

// ✅ GOOD: Specific entitlement
let writeRef = &resource as auth(Mutate) &Resource  // Can mutate

// ❌ LESS IDEAL: Over-privileged
let adminRef = &resource as auth(Owner, Admin, Mutate, Delete) &Resource  // Too many
```

## AI Agent Generation Rules

### Rule 1: Generate References for Borrowing

**Use references when borrowing from storage.**

```cadence
// Generate borrow pattern
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        let vaultRef = signer.storage
            .borrow<&Vault>(from: /storage/vault)
            ?? panic("Vault not found")

        // Use reference
        log(vaultRef.balance)
    }
}
```

### Rule 2: Use Capabilities for Persistence

**Don't generate code that stores references.**

```cadence
// ✅ Generate capability storage
access(all) resource Container {
    access(self) let cap: Capability<&Vault>

    init(cap: Capability<&Vault>) {
        self.cap = cap
    }
}

// ❌ Don't generate reference storage
access(all) resource Container {
    access(self) let ref: &Vault  // COMPILE ERROR
}
```

### Rule 3: Generate Appropriate Entitlements

**Match entitlements to required operations.**

```cadence
// Read-only - no entitlements
let readRef = signer.storage.borrow<&Vault>(from: /storage/vault)

// Write access - specific entitlement
let writeRef = signer.storage
    .borrow<auth(FungibleToken.Withdraw) &Vault>(from: /storage/vault)
```

### Rule 4: Generate Safe Reference Patterns

**Ensure references don't outlive referenced values.**

```cadence
// ✅ Generate scoped usage
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        if let ref = signer.storage.borrow<&Vault>(from: /storage/vault) {
            // Use within scope
            log(ref.balance)
        }
        // Reference invalid after scope
    }
}
```

## Testing References

```cadence
import Test

access(all) fun testReferenceCreation() {
    let resource <- createResource()

    // Create reference
    let ref = &resource as &Resource
    Test.expect(ref != nil, Test.equal(true))

    // Use reference
    let value = ref.getValue()
    Test.expect(value, Test.equal(expectedValue))

    destroy resource
}

access(all) fun testAuthorizedReference() {
    let vault <- createVault(balance: 100.0)

    // Authorized reference
    let authRef = &vault as auth(Withdraw) &Vault

    // Test entitled operation
    let withdrawn <- authRef.withdraw(amount: 10.0)
    Test.expect(withdrawn.balance, Test.equal(10.0))

    destroy withdrawn
    destroy vault
}

access(all) fun testReferenceValidity() {
    let resource <- createResource()
    let ref = &resource as &Resource

    // Valid before move
    Test.expect(ref.isValid(), Test.equal(true))

    destroy resource

    // Invalid after destroy
    Test.expectFailure(fun() {
        ref.someFunction()  // Should fail
    })
}
```

## Common Reference Use Cases

### Use Case 1: Reading Without Moving

```cadence
let resource <- createResource()

// Read data without moving resource
let ref = &resource as &Resource
let data = ref.getData()

// Resource still owned
destroy resource
```

### Use Case 2: Borrowing from Storage

```cadence
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        // Most common pattern
        let ref = signer.storage.borrow<&Vault>(from: /storage/vault)
            ?? panic("Not found")

        log(ref.balance)
    }
}
```

### Use Case 3: Capability Borrowing

```cadence
let cap = getAccount(address).capabilities.get<&{VaultPublic}>(/public/vault)

if let ref = cap.borrow() {
    ref.deposit(from: <-tokens)
}
```

### Use Case 4: Interface References for Restriction

```cadence
// Full resource
access(all) resource Vault {
    access(all) fun publicMethod() { }
    access(Admin) fun adminMethod() { }
}

// Reference to interface hides admin method
let publicRef: &{PublicInterface} = &vault as &{PublicInterface}
publicRef.publicMethod()  // OK
// publicRef.adminMethod()  // COMPILE ERROR: not in interface
```

## Reference Links

- [Cadence References Documentation](https://cadence-lang.org/docs/language/references)
- [Authorized References](https://cadence-lang.org/docs/language/references#authorized-references)
- [Reference Validity](https://cadence-lang.org/docs/language/references#reference-validity)
- [Capabilities Documentation](https://cadence-lang.org/docs/language/capabilities)
