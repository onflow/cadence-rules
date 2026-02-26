# Cadence Code Generation Rules - Index

> **Purpose**: Central navigation hub for all Cadence code generation rules. These rules guide AI agents to generate secure, correct, and idiomatic Cadence code.

## Quick Links

### Core Security Rules (Previously Created)
- **[Access Control and Entitlements](./access-control-and-entitlements.md)** - Secure-by-default access modifiers, entitlements, and visibility rules
- **[Capabilities and Security](./capabilities-and-security.md)** - Comprehensive capability management and security patterns

### Language Fundamentals (High Priority)
- **[Imports](./imports.md)** - Import patterns, network configuration, and flow.json setup
- **[Resources](./resources.md)** - Resource lifecycle, move operator, and linear typing
- **[Transactions](./transactions.md)** - Four-phase structure, entitlements, and transaction patterns
- **[Contracts](./contracts.md)** - Contract structure, initialization, and lifecycle management
- **[Interfaces](./interfaces.md)** - Interface types, conformance, and intersection types

### Advanced Concepts (Medium Priority)
- **[Accounts](./accounts.md)** - Storage, capabilities, keys, contracts, and inbox management
- **[References](./references.md)** - Reference creation, entitlements, and validity
- **[Pre and Post Conditions](./pre-and-post-conditions.md)** - Validation patterns and contract enforcement

### Best Practices
- **[Security Best Practices](./security-best-practices.md)** - Comprehensive security guidelines
- **[Anti-Patterns](./anti-patterns.md)** - Common mistakes to avoid
- **[Design Patterns](./design-patterns.md)** - Proven patterns for maintainability and efficiency

## Rule Priority Levels

### 🔴 Critical (MUST Follow)
Rules that prevent security vulnerabilities or compilation errors:
- Default to private access (`access(self)` or `access(contract)`)
- Use entitlements for privileged operations
- Resources must be explicitly moved or destroyed
- Never expose full resource access via public capabilities
- Always check capabilities before borrowing
- String imports for portability
- Four-phase transaction structure (prepare → pre → execute → post)

### 🟡 High Priority (SHOULD Follow)
Rules that ensure code quality and maintainability:
- Use interface types when publishing capabilities
- Store capability controllers for revocation
- Borrow instead of load/save for efficiency
- Named constants for repeated values
- Descriptive naming conventions
- Minimal transaction entitlements
- Contract initializers for setup

### 🟢 Recommended (NICE to Follow)
Rules that improve developer experience:
- Add explanatory comments
- Regular capability audits
- Comprehensive error messages
- Post-conditions for transaction verification
- Report structs for script accessibility

## Key Principles

### 1. Secure by Default
**Start restrictive, explicitly grant access**
- All declarations begin with `access(self)` or `access(contract)`
- Only make public (`access(all)`) when required
- Use entitlements liberally for privileged operations
- Never trust external data without validation

### 2. Resource Safety
**Linear typing prevents duplication and loss**
- Resources exist in exactly one location
- Must be explicitly moved (`<-`) or destroyed
- Cannot be copied or lost
- Tracked through ownership

### 3. Capability-Based Security
**Leverage Flow's security model**
- Issue minimal-privilege capabilities
- Use entitlements to restrict capability access
- Retain controllers for revocation
- Regular audits and cleanup

### 4. Explicit Over Implicit
**Force developers to make security decisions**
- AI generates restrictive code
- Developer must explicitly relax restrictions
- Clear intent through explicit declarations
- No magic or hidden behavior

## Common Use Cases

### Use Case 1: Token Contract

```cadence
import "FungibleToken"

access(all) contract MyToken {
    // Named constant
    access(all) let TOTAL_SUPPLY: UFix64

    // Resource with entitlements
    access(all) resource Vault {
        access(self) var balance: UFix64

        access(all) view fun getBalance(): UFix64 {
            return self.balance
        }

        access(all) fun deposit(from: @{FungibleToken.Vault}) {
            // Public deposit
        }

        access(Withdraw) fun withdraw(amount: UFix64): @Vault {
            // Entitled withdrawal
        }
    }

    init() {
        self.TOTAL_SUPPLY = 1000000.0
    }
}
```

### Use Case 2: NFT Contract with Interface

```cadence
import "NonFungibleToken"

access(all) contract MyNFT {
    // Interface for public access
    access(all) resource interface NFTPublic {
        access(all) let id: UInt64
        access(all) fun getMetadata(): {String: String}
    }

    // Resource implementing interface
    access(all) resource NFT: NFTPublic {
        access(all) let id: UInt64
        access(self) let metadata: {String: String}

        access(all) fun getMetadata(): {String: String} {
            return self.metadata
        }

        init(id: UInt64, metadata: {String: String}) {
            self.id = id
            self.metadata = metadata
        }
    }

    // Collection for managing NFTs
    access(all) resource Collection {
        access(self) var items: @{UInt64: NFT}

        access(all) fun deposit(nft: @NFT) {
            let id = nft.id
            let old <- self.items[id] <- nft
            destroy old
        }

        access(all) fun withdraw(id: UInt64): @NFT {
            return <- self.items.remove(key: id)
                ?? panic("NFT with ID \(id) not found in collection")
        }

        access(all) fun borrowNFT(id: UInt64): &{NFTPublic}? {
            return &self.items[id] as &{NFTPublic}?
        }

        init() {
            self.items <- {}
        }

        destroy() {
            destroy self.items
        }
    }
}
```

### Use Case 3: Transaction with All Phases

```cadence
import "FungibleToken"

transaction(amount: UFix64, recipient: Address) {
    let senderVault: &{FungibleToken.Provider}
    let recipientVault: &{FungibleToken.Receiver}
    let startBalance: UFix64

    prepare(signer: auth(BorrowValue) &Account) {
        self.senderVault = signer.storage
            .borrow<&{FungibleToken.Provider}>(from: /storage/vault)
            ?? panic("Could not borrow FungibleToken Provider reference from /storage/vault")

        self.recipientVault = getAccount(recipient)
            .capabilities.borrow<&{FungibleToken.Receiver}>(/public/receiver)
            ?? panic("Recipient not found")

        self.startBalance = self.senderVault.balance
    }

    pre {
        amount > 0.0: "Amount must be positive"
        self.senderVault.balance >= amount: "Insufficient balance"
    }

    execute {
        let vault <- self.senderVault.withdraw(amount: amount)
        self.recipientVault.deposit(from: <-vault)
    }

    post {
        self.senderVault.balance == self.startBalance - amount:
            "Sender balance incorrect"
    }
}
```

## AI Agent Guidelines Summary

### When Generating Contracts
- Start all fields with `access(self)` or `access(contract)`
- Use entitlements for privileged functions
- Include `init()` function
- Define public interfaces for external access
- Use named constants for repeated values

### When Generating Resources
- Always use `@` prefix for types
- Always use `<-` operator for operations
- Ensure all code paths handle resources
- Include `destroy()` for nested resources
- Use interfaces to separate public and private operations

### When Generating Transactions
- Use explicit phase structure: `prepare` → `pre` → `execute` → `post`
- Request minimal entitlements in `prepare`
- Validate inputs in `pre`
- Business logic in `execute`
- Verify outcomes in `post`

### When Generating Capabilities
- Issue with minimal entitlements
- Check for existing capabilities first
- Use interface types when publishing
- Store controllers for revocation
- Tag with descriptive metadata

### When Generating Functions
- Default to `access(self)` or `access(contract)`
- Use `access(all) view` for read-only operations
- Use entitlements for privileged operations
- Add pre-conditions for validation
- Add post-conditions for verification

## Rule Interactions

### Security Rules Build On Each Other
1. **Access Control** → Controls who can call functions
2. **Capabilities** → Delegate specific access rights
3. **Entitlements** → Fine-grained permission control
4. **Resources** → Asset ownership and safety
5. **References** → Non-owning access with entitlements

### Pattern Layering
1. **Contracts** define structure
2. **Interfaces** define behavior contracts
3. **Resources** implement owned assets
4. **Capabilities** enable delegation
5. **Transactions** coordinate operations

## Learning Path for Developers

### Level 1: Fundamentals
1. [Imports](./imports.md) - How to import contracts
2. [Contracts](./contracts.md) - Contract structure
3. [Resources](./resources.md) - Understanding resources
4. [Transactions](./transactions.md) - Transaction phases

### Level 2: Security
5. [Access Control and Entitlements](./access-control-and-entitlements.md) - Security model
6. [Capabilities and Security](./capabilities-and-security.md) - Delegation model
7. [Security Best Practices](./security-best-practices.md) - Comprehensive security
8. [Anti-Patterns](./anti-patterns.md) - What to avoid

### Level 3: Advanced
9. [Interfaces](./interfaces.md) - Behavioral contracts
10. [Accounts](./accounts.md) - Account management
11. [References](./references.md) - Non-owning access
12. [Pre and Post Conditions](./pre-and-post-conditions.md) - Validation

### Level 4: Mastery
13. [Design Patterns](./design-patterns.md) - Proven solutions
14. Review all rules for integration
15. Practice with real-world examples

## Quick Reference Cards

### Access Modifiers
```
access(self)         → Private to current scope
access(contract)     → Contract-level access
access(account)      → Account-level access
access(Entitlement)  → Requires specific entitlement
access(all)          → Public access (use sparingly!)
```

### Resource Operations
```
create Resource()    → Create resource
<-                   → Move operator
destroy              → Explicitly destroy
&resource            → Create reference
```

### Transaction Phases
```
prepare   → Access signing accounts (requires entitlements)
pre       → Validate inputs (before execute)
execute   → Main business logic (no account access)
post      → Verify outcomes (after execute)
```

### Storage Operations
```
storage.save()       → Save resource
storage.load()       → Load and remove resource
storage.borrow()     → Borrow reference (preferred)
storage.copy()       → Copy struct value
```

### Capability Operations
```
capabilities.storage.issue()    → Create storage capability
capabilities.publish()          → Make capability public
capabilities.get()              → Get published capability
capability.borrow()             → Borrow reference
capability.check()              → Verify validity
```

### Common Entitlements
```
// Account
auth(BorrowValue) &Account
auth(SaveValue) &Account
auth(LoadValue) &Account
auth(IssueStorageCapabilityController) &Account

// Resources
auth(FungibleToken.Withdraw) &Vault
auth(Owner) &Resource
auth(Admin) &AdminResource
```

## Rule Updates and Maintenance

These rules are based on:
- [Cadence Language Documentation](https://cadence-lang.org/docs/language)
- [Flow Blockchain Documentation](https://developers.flow.com/)
- [Cadence Security Best Practices](https://cadence-lang.org/docs/security-best-practices)
- [Cadence Design Patterns](https://cadence-lang.org/docs/design-patterns)
- [Cadence Anti-Patterns](https://cadence-lang.org/docs/anti-patterns)

**Last Updated**: December 2024

## Support and Questions

For questions about these rules or Cadence development:
- [Flow Discord](https://discord.gg/flow)
- [Cadence Documentation](https://cadence-lang.org/)
- [Flow Forum](https://forum.flow.com/)
- [Flow Stack Overflow](https://stackoverflow.com/questions/tagged/flow-blockchain)

## Complete Rule Index

1. ✅ [Access Control and Entitlements](./access-control-and-entitlements.md)
2. ✅ [Capabilities and Security](./capabilities-and-security.md)
3. ✅ [Imports](./imports.md)
4. ✅ [Resources](./resources.md)
5. ✅ [Transactions](./transactions.md)
6. ✅ [Contracts](./contracts.md)
7. ✅ [Interfaces](./interfaces.md)
8. ✅ [Accounts](./accounts.md)
9. ✅ [References](./references.md)
10. ✅ [Pre and Post Conditions](./pre-and-post-conditions.md)
11. ✅ [Security Best Practices](./security-best-practices.md)
12. ✅ [Anti-Patterns](./anti-patterns.md)
13. ✅ [Design Patterns](./design-patterns.md)

**Total: 13 comprehensive Cadence rules**
