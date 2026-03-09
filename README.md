# Claude Code Skills: Crypto / DeFi

Skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) focused on crypto and DeFi development. Built from real production experience -- debugging live transactions, losing funds to bridge failures, and iterating through dozens of on-chain edge cases.

## Skills

| Skill | Description |
|---|---|
| [deBridge Integration](skills/debridge-integration/SKILL.md) | Cross-chain bridging via deBridge DLN API. 12 critical gotchas, fee structures, order lifecycle, and a complete integration checklist. |
| [Solana Browser Wallet](skills/solana-browser-wallet/SKILL.md) | Ephemeral browser-based Solana wallet patterns. In-memory key management, encrypted IndexedDB vault, recovery flows, and security checklist. |
| [Crypto Contract Design](skills/crypto-contract-design/SKILL.md) | Off-chain logic + on-chain settlement pattern. Two-contract architecture (Platform + FeeVault), backend attestation, emergency timeouts, and Foundry test scenarios. |
| [White-Label Bridge](skills/white-label-bridge/SKILL.md) | Standalone bridge widget with platform fee collection. Fee-first architecture, live quotes, branding removal checklist, and error handling patterns. |

## Installation

### As a Claude Code Plugin (recommended)

```bash
# From your project directory
claude mcp add-skill /path/to/claude-skills/skills/debridge-integration
```

### Manual Installation

Copy skill directories into your Claude Code skills location:

```bash
cp -r skills/debridge-integration ~/.claude/commands/
# or wherever your Claude Code instance loads skills from
```

### Direct Reference

Reference skills in your `CLAUDE.md` or slash commands:

```markdown
allowed-tools: debridge-integration, solana-browser-wallet
```

## How These Were Built

These skills are distilled from:

- **50+ hours** of Claude Code sessions across 3 crypto projects
- **13+ bridge transaction bugs** debugged on mainnet (yes, real money was lost)
- **12 deBridge gotchas** discovered through trial and error
- **3 wallet architecture iterations** (localStorage -> in-memory -> encrypted IndexedDB vault)
- **2 smart contract designs** (game settlement + fee vaults)

Each skill captures the patterns that worked and the mistakes to avoid, so you don't have to learn them the hard way.

## Contributing

Found a gotcha that isn't documented? Open a PR. These skills get better with more battle scars.

## License

MIT
