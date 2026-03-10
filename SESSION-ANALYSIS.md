# Session Analysis -- March 10, 2026

Scraped all Claude Code sessions across 6 projects to identify patterns, recurring tasks, and opportunities for skills/plugins/agents/CLAUDE.md rules.

## Sessions Inventory

| Project | Sessions | Key Activity |
|---|---|---|
| Bundler | 3 | Full Solana token bundler: wallet gen, deBridge bridging, Jupiter swaps, stealth mode, standalone bridge UI |
| Bankr Tracker | 5 | Monopoly casino smart contract design (Base), pitch deck for pump.fun hackathon, autoresearch eval, security questions |
| AgentMoney | 1 | Autonomous agent with $8 USDC: crypto alpha reports, 6 content iterations, pivoting to Twitter + pump.fun token |
| Root/Misc | 1 | Setup/test |

## User Workflow Patterns

1. **Slash command pipeline**: `/investigate` -> `/write-plan` -> `/execute-plan` -> `/fix-all-issues` -> `/merge` -> `/cleanup`
2. **Fast approvals**: "yea", "1", "go for it", "mhm" -- expects autonomous execution after green light
3. **Error debugging via paste**: Pastes console errors and expects immediate root-cause analysis, not "let me investigate"
4. **Multi-device**: Works across Mac Mini, M1 MacBook, and other PCs. Needs seamless repo handoff
5. **Private-first**: Builds in private repos, security reviews before any public push
6. **Context exhaustion**: Long sessions blow through context windows. Needs good continuation summaries

## User Communication Style

- Casual: "lets gooo bro", "fam", "ur", "wtf"
- Impatient with slowness: "why are u taking so long"
- Hates AI-sounding output: "hella AI", "emdashes and hashtags"
- Gives business context naturally in conversation (runs dome.run, works with Rishi + Corey)
- Pushes for quality: will reject 5 iterations until it's right

## Recurring Technical Issues

1. **Node 18 vs 20**: Every project hits this. Default is 18, everything needs 20
2. **deBridge failures**: Lost real money. Bridge errors, wrong terminal states, fee miscalculations
3. **Context window exhaustion**: Bundler session had 4+ continuations
4. **Subagent permissions**: Parallel agents can't run bash, causing verification gaps

## Gap Analysis: What's Missing

### New Skills Needed

| Skill | Why | Priority |
|---|---|---|
| `stealth-bundle` | Staggered multi-wallet execution with configurable jitter. Core pattern from bundler v2 | HIGH |
| `crypto-alpha-writer` | Voice-controlled alpha generation: scrape sources, write in natural degen voice, avoid AI tells | MEDIUM |
| `wallet-safety-check` | Audit codebase for key exposure, env leaks, unsafe git history | HIGH |
| `pump-fun-token-launch` | Token creation + liquidity on pump.fun via Solana. User wants this for agentmoney | MEDIUM |

### Plugins (MCP Servers) Needed

| Plugin | Why |
|---|---|
| Jupiter Swap MCP | Wraps Jupiter with proper reserve calcs (10M lamports for Token-2022), quote fetching, swap execution |
| Solscan/Basescan Explorer MCP | Look up wallets/txs during debugging instead of manual browser checks |
| Pump.fun Data MCP | Token launch data, holder distributions, liquidity events for alpha research |

### Agents Needed

| Agent | Why |
|---|---|
| Crypto Alpha Research | Autonomous agent with USDC budget: scrapes Twitter/on-chain data, synthesizes, publishes. Prototyped in agentmoney session |
| Bridge Transaction Monitor | Background polling of deBridge order status. Alerts on complete/fail/stuck. Would have prevented the "waiting for bridge that already bridged" bug |
| Multi-Wallet Recovery | When pipelines fail mid-execution, assess wallet states across chains and recover scattered funds |

### CLAUDE.md Rules Needed

```markdown
# Node Version
Always run `source ~/.nvm/nvm.sh && nvm use 20` before any npm/npx/node command.

# Wallet Safety
Never log or write private keys to any file that could be committed. Before any git push, verify no secrets in staged files.

# Content Voice
When generating crypto content: no emdashes, no hashtags, no bullet-heavy formatting, no "here's what you need to know." Casual first-person voice.

# Error Response
When user pastes an error, immediately identify root cause. For on-chain errors, decode program error codes. Never say "let me investigate" -- just investigate.

# Context Continuations
Continuation summaries must include: all modified file paths, current task/batch status, pending user requests, wallet addresses and balances at stake.
```

## Existing Skills Coverage

| Existing Skill | Covers | Gaps |
|---|---|---|
| debridge-integration | Bridge API, fees, gotchas | No stealth/jitter patterns |
| solana-browser-wallet | Key management, IndexedDB vault | No multi-wallet orchestration |
| white-label-bridge | Fee collection, branding | Complete for its scope |
| multi-chain-wallet-integration | MetaMask + Phantom + Solflare | Complete for its scope |
| crypto-contract-design | Off-chain + on-chain settlement | Could add Base-specific patterns |
| solana-anchor-deployment | Anchor lifecycle, PDAs | Complete for its scope |
| crypto-refund-system | Cancellation, claims, recovery | No cross-chain refund patterns |
| socket-io-game-state | Multiplayer rooms, turns, disconnect | Complete for its scope |
