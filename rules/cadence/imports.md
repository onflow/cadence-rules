# Cadence Import Rules

> **Purpose**: Define clear guidelines for importing contracts, types, and declarations in Cadence. Proper import patterns ensure code portability across networks and development environments.

## Core Philosophy: Environment-Aware Imports

Cadence supports **three import methods**, each suited for different development contexts. Choose the method that matches your tooling and deployment strategy.

## Three Import Methods

### Method 1: String Imports (Recommended - Flow CLI)

**Use string literals to import by contract name**, with automatic address resolution via `flow.json`:

```cadence
import "FungibleToken"
import "FlowToken"
import "DeFiActions"
import "MyContract"
```

**When to Use**:
- Working with Flow CLI
- Multi-network deployments (testnet, mainnet, emulator)
- Professional contract development
- When using `flow.json` configuration

**How it Works**:
- Flow CLI reads `flow.json` to resolve contract names to addresses
- Different addresses per network (mainnet, testnet, emulator)
- Automatic resolution based on current network context

**Example `flow.json` Configuration**:
```json
{
  "contracts": {
    "FungibleToken": {
      "source": "./cadence/contracts/FungibleToken.cdc",
      "aliases": {
        "emulator": "0xee82856bf20e2aa6",
        "testnet": "0x9a0766d93b6608b7",
        "mainnet": "0xf233dcee88fe0abe"
      }
    },
    "MyContract": {
      "source": "./cadence/contracts/MyContract.cdc",
      "aliases": {
        "emulator": "0xf8d6e0586b0a20c7"
      }
    }
  }
}
```

**Benefits**:
- Network-agnostic code
- Single source of truth for addresses
- Easy network switching
- No code changes when deploying to different networks

### Method 2: Address Imports (Flow Playground/Runner)

**Import all public declarations from an account address**:

```cadence
import 0x1234567890abcdef
```

**When to Use**:
- Flow Playground
- Quick testing and prototyping
- One-off scripts
- When you don't have access to Flow CLI tooling

**Characteristics**:
- Imports ALL public declarations from the specified account
- No contract name filtering
- Direct address specification
- Network-specific (addresses differ per network)

**Limitations**:
- Not portable across networks
- Requires manual address updates
- Can import unintended declarations
- Less explicit about dependencies

### Method 3: Selective Imports with `from` Keyword

**Import specific declarations by name** from a file or address:

```cadence
// From local file
import MyContract from "./contracts/MyContract.cdc"

// From external account
import FungibleToken from 0x1234567890abcdef

// Multiple specific imports
import FungibleToken, NonFungibleToken from 0x1234567890abcdef
```

**When to Use**:
- Need explicit control over imported declarations
- Avoiding namespace pollution
- Clear documentation of dependencies
- Local file imports during development

**Benefits**:
- Explicit about what you're importing
- Reduces namespace conflicts
- Clear dependency tracking
- Can import from files or addresses

## AI Agent Generation Rules

### Rule 1: Default to String Imports

**Always generate string imports by default** unless explicitly instructed otherwise:

```cadence
// ✅ CORRECT: Default pattern
import "FungibleToken"
import "FlowToken"
import "DeFiActions"

// ❌ AVOID: Address imports (unless in Playground)
import 0x1234567890abcdef
```

**Rationale**: String imports work with Flow CLI, which is the standard professional tooling.

### Rule 2: Standard Import Order

**Organize imports in a consistent order**:

1. Core Flow contracts (alphabetical)
2. Third-party protocol contracts (alphabetical)
3. Project contracts (alphabetical)

```cadence
// ✅ CORRECT: Organized imports
// Core Flow contracts
import "FungibleToken"
import "NonFungibleToken"
import "MetadataViews"

// Third-party protocols
import "DeFiActions"
import "SwapConnectors"
import "Staking"

// Project-specific
import "MyContract"
import "MyHelpers"

// ❌ WRONG: Random order
import "MyContract"
import "FungibleToken"
import "DeFiActions"
import "FlowToken"
```

### Rule 3: One Import Per Line

**Never combine multiple imports on one line**:

```cadence
// ✅ CORRECT: One per line
import "FungibleToken"
import "FlowToken"

// ❌ WRONG: Multiple imports
import "FungibleToken", "FlowToken"
```

### Rule 4: No Unused Imports

**Only import what you actually use**:

```cadence
// ✅ CORRECT: Only needed imports
import "FungibleToken"

transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
    }
}

// ❌ WRONG: Unused imports
import "FungibleToken"
import "NonFungibleToken"  // Not used in transaction
import "MetadataViews"     // Not used in transaction

transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let vault = signer.storage.borrow<&{FungibleToken.Vault}>(from: /storage/vault)
    }
}
```

## Import Patterns by File Type

### Contract Files

```cadence
import "FungibleToken"
import "NonFungibleToken"

access(all) contract MyNFTContract {
    // Contract implementation
}
```

### Transaction Files

```cadence
import "FungibleToken"
import "MyContract"

transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        // Transaction logic
    }
}
```

### Script Files

```cadence
import "FungibleToken"
import "MyContract"

access(all) fun main(address: Address): UFix64 {
    // Script logic
}
```

## Network-Specific Considerations

### Emulator Development

String imports work seamlessly with emulator when configured in `flow.json`:

```cadence
import "FungibleToken"  // Resolves to emulator address
import "MyContract"     // Resolves to emulator address
```

### Testnet Deployment

Same import statements, different address resolution:

```cadence
import "FungibleToken"  // Resolves to testnet address
import "MyContract"     // Resolves to testnet address
```

### Mainnet Deployment

Same import statements, different address resolution:

```cadence
import "FungibleToken"  // Resolves to mainnet address
import "MyContract"     // Resolves to mainnet address
```

**No code changes required** when moving between networks!

## Common Import Patterns

### Pattern 1: Standard Token Transaction

```cadence
import "FungibleToken"
import "FlowToken"

transaction(amount: UFix64, to: Address) {
    prepare(signer: auth(BorrowValue) &Account) {
        // Implementation
    }
}
```

### Pattern 2: DeFi Protocol Integration

```cadence
import "FungibleToken"
import "DeFiActions"
import "SwapConnectors"
import "IncrementFiStakingConnectors"

transaction(pid: UInt64) {
    prepare(acct: auth(BorrowValue, SaveValue) &Account) {
        // Implementation
    }
}
```

### Pattern 3: NFT Operations

```cadence
import "NonFungibleToken"
import "MetadataViews"
import "MyNFTCollection"

transaction(nftID: UInt64) {
    prepare(signer: auth(BorrowValue) &Account) {
        // Implementation
    }
}
```

### Pattern 4: Multi-Contract Dependencies

```cadence
// Core contracts
import "FungibleToken"
import "NonFungibleToken"

// Protocol contracts
import "DeFiActions"
import "SwapConnectors"

// Project contracts
import "MyVault"
import "MyNFT"
import "MyHelpers"

access(all) contract MyProtocol {
    // Implementation
}
```

## Import Resolution and Debugging

### Verifying Import Resolution

Use Flow CLI to verify imports resolve correctly:

```bash
# Check contract deployment
flow accounts get 0xAddress --network testnet

# Verify contract exists at address
flow scripts execute ./scripts/check_contract.cdc --network testnet
```

### Common Import Errors

**Error 1: Contract Not Found**
```
error: cannot find contract in imported address
```

**Solution**: Verify `flow.json` has correct address for the network:
- Check network name matches (mainnet/testnet/emulator)
- Verify contract is deployed at specified address
- Ensure contract name matches exactly

**Error 2: Import Cycle**
```
error: cyclic import
```

**Solution**: Restructure contracts to avoid circular dependencies:
- Extract shared interfaces into separate contracts
- Use interface imports instead of concrete types
- Refactor to create clear dependency hierarchy

**Error 3: Ambiguous Import**
```
error: ambiguous use of imported declaration
```

**Solution**: Use fully qualified names:
```cadence
// If two contracts export same name
let value = ContractA.SharedType()
let other = ContractB.SharedType()
```

## flow.json Configuration Best Practices

### Complete Configuration Example

```json
{
  "contracts": {
    "FungibleToken": {
      "source": "./cadence/contracts/FungibleToken.cdc",
      "aliases": {
        "emulator": "0xee82856bf20e2aa6",
        "testnet": "0x9a0766d93b6608b7",
        "mainnet": "0xf233dcee88fe0abe"
      }
    },
    "FlowToken": {
      "source": "./cadence/contracts/FlowToken.cdc",
      "aliases": {
        "emulator": "0x0ae53cb6e3f42a79",
        "testnet": "0x7e60df042a9c0868",
        "mainnet": "0x1654653399040a61"
      }
    },
    "MyContract": {
      "source": "./cadence/contracts/MyContract.cdc",
      "aliases": {
        "emulator": "0xf8d6e0586b0a20c7",
        "testnet": "0x1234567890abcdef"
      }
    }
  },
  "networks": {
    "emulator": "127.0.0.1:3569",
    "testnet": "access.devnet.nodes.onflow.org:9000",
    "mainnet": "access.mainnet.nodes.onflow.org:9000"
  },
  "accounts": {
    "emulator-account": {
      "address": "0xf8d6e0586b0a20c7",
      "key": "..."
    },
    "testnet-account": {
      "address": "0x1234567890abcdef",
      "key": "..."
    }
  }
}
```

### Configuration Checklist

- [ ] All imported contracts listed in `contracts` section
- [ ] Source paths correct for local contracts
- [ ] Aliases provided for all target networks
- [ ] Network endpoints configured
- [ ] Account addresses and keys set up
- [ ] Contract names match import statements exactly

## Testing Import Resolution

### Test Script Example

```cadence
import "FungibleToken"
import "MyContract"

// Verify imports work
access(all) fun main(): Bool {
    log("FungibleToken imported successfully")
    log("MyContract imported successfully")
    return true
}
```

Run with:
```bash
flow scripts execute ./scripts/test_imports.cdc --network emulator
flow scripts execute ./scripts/test_imports.cdc --network testnet
```

## Advanced: Selective Imports for Optimization

When you only need specific declarations:

```cadence
// Import only what you need
import MyContract from "./contracts/MyContract.cdc"

// Instead of importing everything
import "./contracts/MyContract.cdc"
```

**Benefits**:
- Clearer dependencies
- Reduced namespace pollution
- Better documentation

**Use When**:
- Contract exports many declarations
- Only need a subset of functionality
- Avoiding naming conflicts

## Migration Guide

### From Address Imports to String Imports

**Before** (Address imports):
```cadence
import 0x1234567890abcdef
import 0xabcdef1234567890

transaction() {
    // Implementation
}
```

**After** (String imports):
```cadence
import "FungibleToken"
import "MyContract"

transaction() {
    // Same implementation
}
```

**Steps**:
1. Identify contracts at each address
2. Add contracts to `flow.json` with aliases
3. Replace address imports with string imports
4. Test on emulator
5. Deploy to testnet/mainnet

## Reference Links

- [Cadence Imports Documentation](https://cadence-lang.org/docs/language/imports)
- [Flow CLI Configuration](https://developers.flow.com/tools/flow-cli/flow.json/configuration)
- [Network Addresses](https://developers.flow.com/build/core-contracts)
