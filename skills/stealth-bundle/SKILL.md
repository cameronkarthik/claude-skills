---
name: stealth-bundle
description: Use when building multi-wallet transaction pipelines that need to avoid detection as coordinated activity. Covers staggered timing, jitter windows, and multi-wallet orchestration patterns for Solana and EVM chains.
version: 0.1.0
tools: Read, Grep, Glob, Bash, Edit, Write
---

# Stealth Bundle Execution

Patterns for executing coordinated multi-wallet transactions without looking coordinated. Built from production bundler that needed to avoid on-chain clustering detection.

## When to Use

- Building multi-wallet buy/sell pipelines
- Any operation where N wallets need to execute similar transactions without appearing linked
- Bridge + swap pipelines across multiple wallets
- Token launch participation across wallet sets

## Core Problem

Naive bundling is trivially detectable:
- All wallets funded from same source at same time
- All wallets execute same swap within same block
- All wallets bridge from same chain within seconds
- Fund flow graph shows clear hub-and-spoke pattern

## Stealth Architecture

### 1. Staggered Bridge Timing

Do NOT bridge all wallets simultaneously. Spread bridge transactions across a configurable window.

```typescript
interface StealthConfig {
  bridgeJitterMinMs: number;   // e.g., 60000 (1 min)
  bridgeJitterMaxMs: number;   // e.g., 900000 (15 min)
  buyJitterMinMs: number;      // e.g., 5000 (5 sec)
  buyJitterMaxMs: number;      // e.g., 180000 (3 min)
  fundingJitterMinMs: number;  // e.g., 30000 (30 sec)
  fundingJitterMaxMs: number;  // e.g., 600000 (10 min)
}
```

### 2. Jitter Implementation

Use cryptographically random delays, not Math.random:

```typescript
import { randomInt } from 'crypto';

function getJitter(minMs: number, maxMs: number): number {
  return randomInt(minMs, maxMs + 1);
}

async function executeWithJitter<T>(
  fn: () => Promise<T>,
  minMs: number,
  maxMs: number
): Promise<T> {
  const delay = getJitter(minMs, maxMs);
  await new Promise(resolve => setTimeout(resolve, delay));
  return fn();
}
```

### 3. Pipeline Execution Order

For N wallets buying a token:

```
Phase 1: Fund wallets (staggered over fundingJitter window)
  Wallet 1 funded at T+0
  Wallet 2 funded at T+random(30s, 10min)
  Wallet 3 funded at T+random(30s, 10min)
  ...

Phase 2: Bridge (staggered over bridgeJitter window)
  Each wallet bridges ONLY after its own funding confirms
  Wallet 1 bridges at T+fund1_confirm+random(1min, 15min)
  Wallet 2 bridges at T+fund2_confirm+random(1min, 15min)
  ...

Phase 3: Buy (staggered over buyJitter window)
  Each wallet buys ONLY after its own bridge confirms
  Wallet 1 buys at T+bridge1_confirm+random(5s, 3min)
  Wallet 2 buys at T+bridge2_confirm+random(5s, 3min)
  ...
```

### 4. Smart Defaults

For casual use (not trying to fool sophisticated analysis):
- Bridge jitter: 1-5 minutes
- Buy jitter: 10-60 seconds
- Funding jitter: 30s-5min

For maximum stealth (avoiding on-chain forensics):
- Bridge jitter: 15-60 minutes
- Buy jitter: 1-10 minutes
- Funding jitter: 5-30 minutes
- Vary amounts slightly per wallet (not exact same amount)
- Use different bridge routes per wallet if possible

### 5. Amount Variance

Don't buy the exact same amount from every wallet:

```typescript
function varyAmount(baseAmount: number, variancePct: number = 0.15): number {
  const variance = baseAmount * variancePct;
  const offset = (Math.random() * 2 - 1) * variance;
  return Math.round(baseAmount + offset);
}
```

## Execution State Machine

Each wallet runs through states independently:

```
PENDING -> FUNDING -> FUNDED -> BRIDGING -> BRIDGED -> BUYING -> BOUGHT -> DONE
                        |           |          |          |
                        v           v          v          v
                     FUND_ERR   BRIDGE_ERR  BRIDGE_ERR  BUY_ERR
```

Critical: each wallet is independent. If wallet 3 fails to bridge, wallets 1, 2, 4, 5 continue. Don't halt the pipeline for one failure.

## Recovery Patterns

When a wallet gets stuck mid-pipeline:
1. Check wallet balance on source chain (did funding arrive?)
2. Check bridge order status (is it pending, fulfilled, or failed?)
3. Check wallet balance on destination chain (did bridge complete but status API lag?)
4. If funds are on destination but swap failed: retry swap only
5. If funds are stuck on bridge: wait for bridge timeout or manual recovery
6. Last resort: consolidate all wallet funds back to source wallet

## Gotchas

1. **Bridge status polling**: deBridge `Fulfilled` is the terminal state. Do NOT wait for `ClaimedUnlock` -- it may never come.
2. **Solana skipPreflight**: Always use `skipPreflight: true` for bridge transactions. Simulation will fail because the bridge program has custom logic.
3. **Gas reserves**: Leave enough for the buy transaction after bridging. For Solana, reserve 10M lamports (0.01 SOL) minimum.
4. **Rate limits**: Jupiter and deBridge have rate limits. Space your quote requests, don't fire all N at once.
5. **Blockhash expiry**: Solana blockhashes expire after ~60 seconds. If using staggered execution, fetch a fresh blockhash for each wallet's transaction, not one shared blockhash.

## Checklist

- [ ] Jitter config is user-adjustable in UI
- [ ] Each wallet operates independently (no shared failure state)
- [ ] Bridge status polling uses DLN endpoint, not Stats API
- [ ] Amount variance applied to avoid identical buy amounts
- [ ] Fresh blockhash fetched per wallet transaction
- [ ] Recovery flow handles each failure state
- [ ] Progress UI shows per-wallet state independently
- [ ] Cancel button stops pending wallets but doesn't abort in-flight ones
