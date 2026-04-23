# AGENTS.md

This file tells AI coding agents (Claude Code, Codex, Cursor, Copilot, Gemini CLI, and
others) how this repository is organized so they can locate and apply the right rules.
It is loaded into agent context on attach — keep it concise.

## Overview

`cadence-rules` is a documentation-only repository of AI-agent rule files for Flow and
Cadence development. Rules are authored as Markdown (`.md`) and Cursor rule (`.mdc`)
files under `rules/`, and are intended to be dropped into a downstream Flow project so
that Cursor (and other agents that read AGENTS.md or `.mdc` files) apply them
automatically. There is no build system, no test suite, and no CI — the repo's
"artifact" is the rule content itself.

## Repository Layout

```
cadence-rules/
├── README.md
├── user-preferences.mdc              # alwaysApply: true — global AI style/philosophy
└── rules/
    ├── AI-DOCUMENTATION-MAINTENANCE-GUIDE.md   # meta-guide for humans; NOT agent context
    ├── cadence/                      # 13 language rule .md + index.md + 1 .mdc
    ├── workflows/                    # 2 .mdc (project/dev workflow rules)
    └── defi-actions/                 # 11 rule .md + index.md + 1 .mdc + workflows/
```

### `rules/cadence/` — Cadence language rules
- `index.md` — navigation hub with priority tiers (Critical / High / Medium)
- `cadence-nft-standards.mdc` — NFT standard conformance, MetadataViews, modular traits
- `access-control-and-entitlements.md`, `capabilities-and-security.md`
- `imports.md`, `resources.md`, `transactions.md`, `contracts.md`, `interfaces.md`
- `accounts.md`, `references.md`, `pre-and-post-conditions.md`
- `security-best-practices.md`, `anti-patterns.md`, `design-patterns.md`

### `rules/workflows/` — project setup and development lifecycle
- `flow-configuration.mdc` — `flow.json`, FCL setup, multi-network deployment
- `flow-development-workflow.mdc` — documentation-first debugging, gas optimization,
  testnet validation

### `rules/defi-actions/` — DeFiActions composition patterns
- `ai-generation-entrypoint.mdc` (**alwaysApply: true**) — canonical entrypoint for
  agents generating DeFiActions transactions; enter here for restake/zapper flows
- `index.md` — navigation hub with quick links
- `core-framework.md`, `type-system.md`, `connectors.md`, `composition.md`,
  `interface-inheritance.md`, `patterns.md`, `safety-rules.md`, `quick-checklist.md`,
  `testing.md`, `transaction-templates.md`, `ai-generation-guide.md`
- `workflows/restaking-workflow.md`, `workflows/autobalancer-workflow.md`

## `.mdc` Frontmatter Convention

Cursor rule files use YAML frontmatter. The repo's convention is a `description` plus
an `alwaysApply` flag, but one file intentionally omits the description. Audit of all
five `.mdc` files:

| File | `description` | `alwaysApply` |
|---|---|---|
| `user-preferences.mdc` | yes | `true` |
| `rules/cadence/cadence-nft-standards.mdc` | yes | `false` |
| `rules/workflows/flow-configuration.mdc` | yes | `false` |
| `rules/workflows/flow-development-workflow.mdc` | yes | `false` |
| `rules/defi-actions/ai-generation-entrypoint.mdc` | **omitted** | `true` |

`ai-generation-entrypoint.mdc` has only `alwaysApply: true` with no description — this
is deliberate (it is always loaded, so description-based matching does not apply).
When editing or adding a new `.mdc`, include both fields unless the file is always-on
and intentionally description-less like the entrypoint. The two `alwaysApply: true`
files (`user-preferences.mdc` and `ai-generation-entrypoint.mdc`) load in every agent
session; all other `.mdc` files set `alwaysApply: false` and are loaded on-demand.
Preserve each file's existing frontmatter shape — do not convert `.mdc` to `.md` or
vice versa.

## How Agents Use This Repo

Downstream Flow projects copy or symlink these files into their own tree so Cursor
picks them up — this repo is the source of truth, not a runtime dependency. The README
documents the intended dev sequence: **emulator → frontend/FCL → testnet → mainnet**,
and rule content assumes that order.

## Conventions and Gotchas

- **Links are relative.** `AI-DOCUMENTATION-MAINTENANCE-GUIDE.md` requires all
  cross-file links to be relative (`./foo.md`, `../bar.md`) so pages stay valid when
  copied into downstream projects.
- **`AI-DOCUMENTATION-MAINTENANCE-GUIDE.md` is a human-only meta-guide.** Its own
  header reads "Do not include in agent context." Do not link to it from rule files
  that agents load.
- **No build, no tests, no lint.** Do not fabricate `npm test` / `make lint` / etc.
  Verification is by reading — changes should preserve Cursor frontmatter, relative
  links, and the `alwaysApply` flag on the two always-on files.
- **Canonical content lives under `rules/`.** `SEO_AUDIT_REPORT.md` and
  `.audit-extract.json` at the repo root are transient audit artifacts on feature
  branches, not part of the ruleset.
- **Cadence 1.0 syntax.** Examples use `access(all)`, entitlements, and string imports
  (e.g., `import "FungibleToken"`) — never addresses in source. Do not regress to
  pre-1.0 patterns (`pub`, address imports).
- **DeFiActions restake invariants** (enforced by the entrypoint and maintenance
  guide): `pid`-only parameters; derive pair token types and `stableMode` from the
  pool; size by `source.minimumAvailable()` / `sink.minimumCapacity()` (no manual
  slippage math); assert `vault.balance == 0.0` before `destroy`.

## Files Not to Modify Casually

- `user-preferences.mdc` — loads in every session; edits change global agent behavior.
- `rules/defi-actions/ai-generation-entrypoint.mdc` — the only always-on DeFiActions
  rule; keep it lean since it enters every context window.
- `rules/AI-DOCUMENTATION-MAINTENANCE-GUIDE.md` — editorial process doc; update only
  when the maintenance workflow itself changes.
