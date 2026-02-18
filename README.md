# Cadence Rules

A comprehensive set of Cursor rules for building on the Flow blockchain with Cadence smart contracts and FCL frontend integration.

## Overview

This repository provides AI-friendly documentation for Flow blockchain development, organized into two layers:

### Root-Level Guides (Quick Reference)
| File | Purpose |
|------|---------|
| [cadence-nft-standards.mdc](./cadence-nft-standards.mdc) | NFT-specific patterns, MetadataViews integration, modular architecture for complex traits |
| [flow-configuration.mdc](./flow-configuration.mdc) | Complete `flow.json` & FCL setup guide covering configuration, deployment, and network management |
| [flow-development-workflow.mdc](./flow-development-workflow.mdc) | Development lifecycle, debugging methodology, gas optimization, testnet validation |
| [user-preferences.mdc](./user-preferences.mdc) | Communication style and development philosophy preferences (alwaysApply: true) |

### Detailed Rules Library
- **[rules/cadence/](./rules/cadence/)** - 13 comprehensive Cadence language rules covering security, syntax, patterns, and best practices
- **[rules/defi-actions/](./rules/defi-actions/)** - DeFi-specific action composition patterns and workflows

## How to Use

### 1. Add to Your Cursor Project
Place these files in your Flow project root directory. Cursor will automatically detect and apply these rules when providing AI assistance.

### 2. Development Workflow
The rules enforce this recommended development sequence:
1. **Setup** → Ensure `flow.json` and FCL config are correct
2. **Emulator** → Test contracts locally first
3. **Frontend Integration** → Test FCL interactions with emulator
4. **Testnet Deployment** → Deploy and validate on testnet
5. **Production** → Deploy to mainnet after comprehensive testing

### 3. Key Benefits
- **Error Prevention**: Proactive guidance on common Flow/Cadence pitfalls
- **Standards Compliance**: Enforces NonFungibleToken interface requirements
- **Full-Stack Coverage**: Spans from Cadence contracts to React/FCL frontend
- **Documentation-First**: Prioritizes official Flow documentation and patterns
- **Modular Architecture**: Advanced patterns for complex, evolving NFT systems
- **Security-Focused**: Comprehensive security practices for production deployments
- **Frontend Integration**: Seamless FCL setup with automated contract address resolution

## Quick Reference

### Common Issues These Rules Address
- ❌ Resource type syntax errors (`@` vs `&` vs `{}`)
- ❌ Transaction authorization mismatches
- ❌ FCL configuration network conflicts
- ❌ Contract deployment verification gaps
- ❌ Computation limit exceeded errors (gas optimization)
- ❌ Interface compliance violations
- ❌ Multi-network configuration inconsistencies
- ❌ Frontend contract address resolution failures
- ❌ Security vulnerabilities in access control and capabilities

### Development Philosophy
- **Documentation-Driven**: Reference official Flow docs first
- **Standards Compliance**: Follow established Flow/Cadence patterns
- **Iterative Testing**: Fix one issue at a time, test frequently
- **Full-Stack Awareness**: Consider contracts → transactions → FCL → UI

## Detailed Rules Library

### Cadence Language Rules ([rules/cadence/](./rules/cadence/))
Comprehensive coverage of Cadence language fundamentals:
- **Security**: Access control, entitlements, capabilities
- **Core Concepts**: Resources, contracts, transactions, interfaces
- **Advanced Topics**: Accounts, references, pre/post conditions
- **Best Practices**: Security patterns, anti-patterns, design patterns

[View complete Cadence rules index →](./rules/cadence/index.md)

### DeFi Action Patterns ([rules/defi-actions/](./rules/defi-actions/))
Specialized rules for DeFi protocol integration:
- Core framework interfaces (Source, Sink, Swapper, AutoBalancer)
- Connector patterns for vault operations and protocol integration
- Workflow templates for restaking and auto-balancing
- Safety rules and testing patterns

[View DeFi actions index →](./rules/defi-actions/index.md)

## Advanced Features

### Modular NFT Architecture
The rules include patterns for complex NFTs with:
- Dynamic trait evolution systems
- Breeding and genetic mechanics
- Lazy trait initialization
- Cross-module interactions

### Gas Optimization
Comprehensive strategies for:
- Accumulative processing logic
- Computation limit management
- Efficient loop patterns
- Batch processing techniques

### Multi-Network Configuration
Environment-based setup for:
- Emulator, testnet, and mainnet configurations
- FCL integration with automatic address resolution
- Network-specific contract aliases
- Secure key management

## Getting Started

1. **Clone this repository** to your Flow project
2. **Use Cursor IDE** for AI-assisted development
3. **Ask Flow/Cadence questions** - AI references these rules automatically
4. **Follow the workflow**: emulator → testnet → mainnet

### Quick Start
```bash
# Initialize Flow project
flow init

# Reference root guides for quick help
# - cadence-nft-standards.mdc for NFT development
# - flow-configuration.mdc for setup and deployment
# - flow-development-workflow.mdc for debugging and optimization

# Explore detailed rules for comprehensive coverage
# - rules/cadence/ for language fundamentals
# - rules/defi-actions/ for DeFi integration patterns
```

## Repository Structure

```
cadence-rules/
├── README.md
├── cadence-nft-standards.mdc          # NFT-specific patterns
├── flow-configuration.mdc             # Setup & deployment guide
├── flow-development-workflow.mdc      # Development lifecycle
├── user-preferences.mdc               # AI behavior preferences
└── rules/
    ├── cadence/                       # 13 comprehensive Cadence rules
    │   ├── index.md                   # Navigation hub
    │   ├── access-control-and-entitlements.md
    │   ├── capabilities-and-security.md
    │   ├── resources.md
    │   ├── transactions.md
    │   └── ... (9 more)
    └── defi-actions/                  # DeFi integration patterns
        ├── index.md                   # Quick links
        ├── core-framework.md
        ├── connectors.md
        └── ... (10 more)
```

The rules automatically guide AI responses to match Flow best practices and prevent common development pitfalls. 