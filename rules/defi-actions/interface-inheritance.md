# Interface Inheritance Cheatsheet

> Purpose: Remove ambiguity for humans and AI about which fields and methods must be implemented when creating new connectors. Covers multi-level interface inheritance used across `DeFiActions`.

## Hierarchy (struct interfaces)

- `IdentifiableStruct`
  - Provides traceability via `uniqueID` and requires introspection methods
- `Source : IdentifiableStruct`
  - Provides tokens (`getSourceType`, `minimumAvailable`, `withdrawAvailable`)
- `Sink : IdentifiableStruct`
  - Accepts tokens (`getSinkType`, `minimumCapacity`, `depositCapacity`)
- `Swapper : IdentifiableStruct`
  - Converts tokens (`inType`, `outType`, `quoteIn`, `quoteOut`, `swap`, `swapBack`)
- `PriceOracle : IdentifiableStruct`
  - Prices assets (`unitOfAccount`, `price`)
- `Flasher : IdentifiableStruct`
  - Flash loans (`borrowType`, `calculateFee`, `flashLoan`)

Note: If your struct implements `Source`, you implicitly must satisfy all required members of `IdentifiableStruct`. You do not need to list `IdentifiableStruct` again in the conformance list; `Source` already extends it.

## Required members by interface

### IdentifiableStruct (always required indirectly)
Implementations MUST include the following members with exactly these shapes (access modifiers are relative to your contract):

```cadence
// Required field
access(contract) var uniqueID: DeFiActions.UniqueIdentifier?

// Required functions
access(all) fun getComponentInfo(): DeFiActions.ComponentInfo
access(contract) view fun copyID(): DeFiActions.UniqueIdentifier?
access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?)
```

Tips:
- Use `access(contract)` for `uniqueID`, `copyID`, and `setID` inside your own contract module. Do not use `pub`.
- `id()` is provided by the interface; you do NOT implement it.

### Source
```cadence
access(all) view fun getSourceType(): Type
access(all) fun minimumAvailable(): UFix64
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault}
```

### Sink
```cadence
access(all) view fun getSinkType(): Type
access(all) fun minimumCapacity(): UFix64
access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
```

### Swapper
```cadence
access(all) view fun inType(): Type
access(all) view fun outType(): Type
access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {DeFiActions.Quote}
access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {DeFiActions.Quote}
access(all) fun swap(quote: {DeFiActions.Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault}
access(all) fun swapBack(quote: {DeFiActions.Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault}
```

## Minimal implementation skeletons

### New Source
```cadence
import "FungibleToken"
import "DeFiActions"

access(all) contract MyProtocolConnectors {
    access(all) struct MySource : DeFiActions.Source {
        // IdentifiableStruct required field
        access(contract) var uniqueID: DeFiActions.UniqueIdentifier?

        // Local fields
        access(self) let type: Type
        access(self) let someCap: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>

        init(someCap: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>, uniqueID: DeFiActions.UniqueIdentifier?) {
            pre { someCap.check(): "Invalid capability" }
            self.uniqueID = uniqueID
            self.someCap = someCap
            self.type = someCap.borrow()!.getType()
        }

        // Source
        access(all) view fun getSourceType(): Type { self.type }
        access(all) fun minimumAvailable(): UFix64 { self.someCap.borrow()?.balance ?? 0.0 }
        access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
            if let ref = self.someCap.borrow() {
                let amount = maxAmount <= ref.balance ? maxAmount : ref.balance
                return <-ref.withdraw(amount: amount)
            }
            return <- DeFiActionsUtils.getEmptyVault(self.type)
        }

        // IdentifiableStruct
        access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
            DeFiActions.ComponentInfo(type: self.getType(), id: self.id(), innerComponents: [])
        }
        access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? { self.uniqueID }
        access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) { self.uniqueID = id }
    }
}
```

### New Sink
```cadence
import "FungibleToken"
import "DeFiActions"

access(all) contract MyProtocolConnectors {
    access(all) struct MySink : DeFiActions.Sink {
        access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
        access(self) let type: Type
        access(self) let receiver: Capability<&{FungibleToken.Vault}>

        init(receiver: Capability<&{FungibleToken.Vault}>, uniqueID: DeFiActions.UniqueIdentifier?) {
            pre { receiver.check(): "Invalid capability" }
            self.uniqueID = uniqueID
            self.receiver = receiver
            self.type = receiver.borrow()!.getType()
        }

        access(all) view fun getSinkType(): Type { self.type }
        access(all) fun minimumCapacity(): UFix64 { self.receiver.check() ? UFix64.max : 0.0 }
        access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
            if let r = self.receiver.borrow() { r.deposit(from: <-from.withdraw(amount: from.balance)) }
        }

        access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
            DeFiActions.ComponentInfo(type: self.getType(), id: self.id(), innerComponents: [])
        }
        access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? { self.uniqueID }
        access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) { self.uniqueID = id }
    }
}
```

### New Swapper
```cadence
import "FungibleToken"
import "DeFiActions"

access(all) contract MyProtocolConnectors {
    access(all) struct MySwapper : DeFiActions.Swapper {
        access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
        access(self) let _in: Type
        access(self) let _out: Type

        init(inType: Type, outType: Type, uniqueID: DeFiActions.UniqueIdentifier?) {
            self._in = inType
            self._out = outType
            self.uniqueID = uniqueID
        }

        access(all) view fun inType(): Type { self._in }
        access(all) view fun outType(): Type { self._out }
        access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {DeFiActions.Quote} { panic("impl") }
        access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {DeFiActions.Quote} { panic("impl") }
        access(all) fun swap(quote: {DeFiActions.Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault} { panic("impl") }
        access(all) fun swapBack(quote: {DeFiActions.Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault} { panic("impl") }

        access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
            DeFiActions.ComponentInfo(type: self.getType(), id: self.id(), innerComponents: [])
        }
        access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? { self.uniqueID }
        access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) { self.uniqueID = id }
    }
}
```

## Common mistakes (and fixes)

- Forgetting `uniqueID` field: add `access(contract) var uniqueID: DeFiActions.UniqueIdentifier?`.
- Wrong visibility: use `access(contract)` for `uniqueID`, `copyID`, and `setID`.
- Missing introspection: always implement `getComponentInfo`, `copyID`, and `setID`.
- Swapper method names: use `inType`/`outType` and `quoteIn`/`quoteOut` with the `reverse` parameter.

## Implementation checklist

- Choose the right base: `Source`, `Sink`, `Swapper`, etc. No need to list `IdentifiableStruct` manually.
- Include `uniqueID` and the three IdentifiableStruct functions.
- Ensure type compatibility preconditions match `DeFiActions` expectations.
- Emit nothing yourself; interface-level events are emitted by base implementations where relevant. 