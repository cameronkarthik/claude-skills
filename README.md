# Claude Code Skills: Crypto / DeFi

Skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) focused on crypto and DeFi development. Built from real production experience -- debugging live transactions, losing funds to bridge failures, and iterating through dozens of on-chain edge cases.

## Skills

### Bridging & Wallets

| Skill | Description |
|---|---|
| [deBridge Integration](skills/debridge-integration/SKILL.md) | Cross-chain bridging via deBridge DLN API. 12 critical gotchas, fee structures, order lifecycle, and a complete integration checklist. |
| [Solana Browser Wallet](skills/solana-browser-wallet/SKILL.md) | Ephemeral browser-based Solana wallet patterns. In-memory key management, encrypted IndexedDB vault, recovery flows, and security checklist. |
| [White-Label Bridge](skills/white-label-bridge/SKILL.md) | Standalone bridge widget with platform fee collection. Fee-first architecture, live quotes, branding removal checklist, and error handling patterns. |
| [Multi-Chain Wallet Integration](skills/multi-chain-wallet-integration/SKILL.md) | Connecting MetaMask + Phantom + Solflare in one app. Signature-based auth (EIP-191 + Ed25519), chain-scoped state, deposit verification, and the 6 wallet bugs you will hit. |

### Smart Contracts & Deployment

| Skill | Description |
|---|---|
| [Crypto Contract Design](skills/crypto-contract-design/SKILL.md) | Off-chain logic + on-chain settlement pattern. Two-contract architecture (Platform + FeeVault), backend attestation, emergency timeouts, and Foundry test scenarios. |
| [Solana Anchor Deployment](skills/solana-anchor-deployment/SKILL.md) | Full Anchor deployment lifecycle. PDA initialization, Ed25519 signing, keypair management, Helius RPC config, and devnet-to-mainnet migration checklist. |
| [Crypto Refund System](skills/crypto-refund-system/SKILL.md) | Complete refund architecture for crypto deposits. Cancellation signing, atomic persistence, claim flows, Solana rent accounting, emergency cancel, and mutual exclusion patterns. |

### Operations & Security

| Skill | Description |
|---|---|
| [Stealth Bundle](skills/stealth-bundle/SKILL.md) | Staggered multi-wallet transaction execution with configurable jitter. Avoids on-chain clustering detection for coordinated operations. |
| [Wallet Safety Check](skills/wallet-safety-check/SKILL.md) | Pre-push security audit for crypto projects. Scans for private key exposure, credential leaks, unsafe storage, and git history contamination. |

### Content & Research

| Skill | Description |
|---|---|
| [Crypto Alpha Writer](skills/crypto-alpha-writer/SKILL.md) | On-chain intelligence report generation targeting CT/degen audience. Data collection via paid APIs, cross-referencing, and natural non-AI voice. |

### Payment & Commerce

| Skill | Description |
|---|---|
| [SolCard Agent](skills/solcard-agent/SKILL.md) | Crypto-to-card payment automation via SolCard's reverse-engineered API. Session management, deposit flow across 9 networks, virtual Visa/Mastercard retrieval, and security rules for autonomous agent purchases. |

### Real-Time Multiplayer

| Skill | Description |
|---|---|
| [Socket.IO Game State](skills/socket-io-game-state/SKILL.md) | Real-time multiplayer game state management. Room lifecycle, deposit verification, turn timers, disconnect/reconnect handling, settlement flows, and stale room cleanup. |

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

- **80+ hours** of Claude Code sessions across 5 crypto projects
- **30+ sessions** building a live multiplayer game with ETH and SOL stakes
- **13+ bridge transaction bugs** debugged on mainnet (yes, real money was lost)
- **12+ refund-related bugs** where real user funds were stuck
- **12 deBridge gotchas** discovered through trial and error
- **3 wallet architecture iterations** (localStorage -> in-memory -> encrypted IndexedDB vault)
- **3 smart contract iterations** (Solidity on Base, Anchor on Solana)
- **Dual-chain deployment** (Base + Solana mainnet) with production traffic
- **6 iterations** on AI-generated crypto content to eliminate AI voice patterns
- **Autonomous agent experiment** with real USDC budget and micropayment APIs

Each skill captures the patterns that survived production and the mistakes to avoid, so you don't have to learn them the hard way.

## Session Analysis

See [SESSION-ANALYSIS.md](SESSION-ANALYSIS.md) for a full breakdown of all Claude Code sessions, workflow patterns, and recommendations for future skills/plugins/agents.

## Contributing

Found a gotcha that isn't documented? Open a PR. These skills get better with more battle scars.

## License

MIT
