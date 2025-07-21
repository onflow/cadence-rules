# Cadence Rules

A comprehensive set of Cursor rules for building on the Flow blockchain with Cadence smart contracts and FCL frontend integration.

## Overview

This repository contains 5 specialized `.mdc` files that serve as Cursor rules to guide AI assistance during Flow development:

| File | Purpose |
|------|---------|
| `cadence-nft-standards.mdc` | NFT development standards, modular architecture patterns, security best practices |
| `cadence-syntax-patterns.mdc` | Language syntax, common pitfalls, debugging strategies, authorization patterns |
| `flow-development-workflow.mdc` | Complete development lifecycle, deployment strategies, FCL integration |
| `flow-project-config.mdc` | `flow.json` management, multi-network configuration, contract deployment |
| `user-preferences.mdc` | Communication style and development philosophy preferences |

## How to Use

### 1. Add to Your Cursor Project
Place these `.mdc` files in your Flow project root directory. Cursor will automatically detect and apply these rules when providing AI assistance.

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

## Quick Reference

### Common Issues These Rules Address
- ❌ Resource type syntax errors (`@` vs `&` vs `{}`)
- ❌ Transaction authorization mismatches  
- ❌ FCL configuration network conflicts
- ❌ Contract deployment verification gaps
- ❌ Computation limit exceeded errors
- ❌ Interface compliance violations

### Development Philosophy
- **Documentation-Driven**: Reference official Flow docs first
- **Standards Compliance**: Follow established Flow/Cadence patterns
- **Iterative Testing**: Fix one issue at a time, test frequently
- **Full-Stack Awareness**: Consider contracts → transactions → FCL → UI

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

## Getting Started

1. Clone or download these `.mdc` files to your Flow project
2. Use Cursor IDE for development
3. Ask questions about Flow/Cadence - the AI will reference these rules
4. Follow the enforced workflow: emulator → testnet → mainnet

The rules automatically guide AI responses to match Flow best practices and prevent common development pitfalls. 