---
name: debridge-integration
description: Use when integrating deBridge DLN for cross-chain bridging, building bridge features, or debugging bridge transaction failures. Covers Solana <-> EVM bridging via the DLN API.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch
---

# deBridge DLN Integration

Hard-won integration guide for deBridge DLN cross-chain bridge API. Built from debugging dozens of live transaction failures across Solana and EVM chains.

## When to Use

- Integrating deBridge DLN into any application
- Building cross-chain bridge features (Solana <-> EVM)
- Debugging bridge transaction failures, reverts, or stuck orders
- Estimating bridge fees for UI display

## Pre-Flight Checklist

Before writing any bridge code, confirm:

1. Which chains? (Solana=7565164, Base=8453, Ethereum=1, Arbitrum=42161, etc.)
2. Which direction? (Solana->EVM and EVM->Solana have DIFFERENT fee models and gotchas)
3. Native token addresses: Solana=`11111111111111111111111111111111`, EVM=`0x0000000000000000000000000000000000000000`
4. Is this browser or server code? (Browser has additional constraints -- see Gotcha #3, #8)

## API Endpoints

| Purpose | Endpoint |
|---|---|
| Create order | `POST https://dln.debridge.finance/v1.0/dln/order/create-tx` |
| Order status (DLN) | `GET https://dln.debridge.finance/v1.0/dln/order/{orderId}/status` |
| Order status (Stats) | `GET https://stats-api.dln.trade/api/Orders/{orderId}/state` |

**Use the DLN endpoint for status polling.** The Stats API uses a different response shape (`state` vs `status` field) and is less reliable for real-time polling.

## Fee Structure

### Solana -> EVM (bridge-out)

| Component | Cost |
|---|---|
| fixFee | ~15,000,000 lamports (0.015 SOL) |
| ATA creation (up to 3) | ~6,200,000 lamports (0.006 SOL) |
| Order state rent | ~2,200,000 lamports (0.002 SOL) |
| **Total with safety margin** | **~35,000,000 lamports (0.035 SOL)** |

### EVM -> Solana (bridge-back)

| Component | Cost |
|---|---|
| Protocol fee | 0.001 ETH (added to tx.value) |
| Gas | Minimal on L2s (~0.0001 ETH) |

## Critical Gotchas (The 12 Rules)

These are ordered by how likely they are to waste your time.

### Transaction Construction

**1. Never use `prependOperatingExpenses` for Solana-source bridges.**
The API parameter causes fees to be deducted from the input amount. For Solana sources, fees come from the output side. Using this flag causes silent fund loss.

**2. `srcChainTokenInAmount: "auto"` is invalid for EVM sources.**
Must query the actual wallet balance and pass the exact amount. Only Solana supports "auto".

**3. deBridge returns hex-encoded VersionedTransaction for Solana.**
The response `tx.data` is `0x`-prefixed hex, NOT base64. Deserialize with pure JS:
```typescript
function hexToBytes(hex: string): Uint8Array {
  const clean = hex.startsWith('0x') ? hex.slice(2) : hex;
  const bytes = new Uint8Array(clean.length / 2);
  for (let i = 0; i < clean.length; i += 2) {
    bytes[i / 2] = parseInt(clean.substring(i, i + 2), 16);
  }
  return bytes;
}
```
Do NOT use `Buffer.from(hex, 'hex')` in browser code.

**4. Refresh blockhash before signing.**
The blockhash embedded in the API response can be stale by the time the user signs. Always fetch a fresh one and replace it in the transaction.

**5. `skipPreflight: true` is required for Solana bridge transactions.**
Address lookup tables in deBridge transactions cause false simulation failures. Always skip preflight.

### Transaction Confirmation

**6. `confirmTransaction` only checks block inclusion, NOT success.**
After confirmation, MUST call `getTransaction()` and check `meta.err`:
```typescript
const tx = await connection.getTransaction(sig, {
  maxSupportedTransactionVersion: 0,
  commitment: 'confirmed'
});
if (tx?.meta?.err) throw new Error(`Transaction failed: ${JSON.stringify(tx.meta.err)}`);
```

**7. Bridge-back `tx.value` = bridgeAmount + protocolFee.**
The wallet needs `value` + gas. If building a "bridge max" feature, subtract both protocol fee AND gas from the available balance.

### Browser-Specific

**8. No `Buffer.from` in browser code.**
Use `Uint8Array`, `TextEncoder`, `atob()` + manual byte conversion. Every `Buffer` usage will break in production browsers without polyfills.

**9. Deprecated `confirmTransaction(signature, commitment)` can hang.**
Use the blockhash form instead:
```typescript
await connection.confirmTransaction({
  signature,
  blockhash,
  lastValidBlockHeight
}, 'confirmed');
```

### Token & Fee Edge Cases

**10. Jupiter swap reserve must be 10M lamports for Token-2022 tokens.**
Breakdown: WSOL ATA ~2.04M + Token-2022 ATA ~2.5M + tx fee 5K + priority fee ~3.5M + Jupiter internal ~2M.

**11. Use single-pass bridge-back, not two-pass.**
deBridge embeds DEX swap parameters (e.g., ETH->WETH->USDC->bridge) in the calldata with price limits. A two-pass approach (discovery call then adjusted call) causes stale swap params and on-chain reverts. Use a single API call with the known 0.001 ETH protocol fee pre-subtracted.

### Development Environment

**12. CSP eval warnings in Next.js dev mode.**
Set `config.devtool = "cheap-module-source-map"` in webpack config to suppress.

## Order Status Lifecycle

```
Created -> Fulfilled -> SentUnlock -> ClaimedUnlock    (success path)
Created -> OrderCancelled -> SentOrderCancel -> ClaimedOrderCancel    (cancel path)
```

**`Fulfilled` = funds delivered to destination.** This is the terminal state for application logic.

`SentUnlock` / `ClaimedUnlock` = solver settlement on the SOURCE chain. Irrelevant to the end user. Do NOT wait for these -- they can take much longer and don't affect fund availability.

### Polling Implementation

```typescript
async function waitForBridgeOrder(orderId: string, timeoutMs = 300_000): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    const res = await fetch(
      `https://dln.debridge.finance/v1.0/dln/order/${orderId}/status`
    );
    const data = await res.json();

    if (data.status === 'Fulfilled') return;
    if (data.status === 'OrderCancelled') throw new Error('Bridge order cancelled');

    await new Promise(r => setTimeout(r, 5_000)); // Poll every 5s
  }
  throw new Error('Bridge order timed out');
}
```

## Integration Checklist

Use this checklist when implementing a new bridge feature:

- [ ] Identify source and destination chains + their chain IDs
- [ ] Calculate fee overhead for the source chain (see Fee Structure above)
- [ ] Build `create-tx` API call with correct parameters
- [ ] Handle `0x`-prefixed hex deserialization for Solana transactions
- [ ] Refresh blockhash before wallet signing
- [ ] Send with `skipPreflight: true` for Solana
- [ ] Confirm with blockhash form (not deprecated string form)
- [ ] Verify `meta.err` after confirmation
- [ ] Extract `orderId` from transaction logs or API response
- [ ] Poll order status using DLN endpoint (not Stats API)
- [ ] Terminate on `Fulfilled` (not `ClaimedUnlock`)
- [ ] Handle cancellation and timeout gracefully
- [ ] For bridge-back: use single-pass with known protocol fee
- [ ] For browser: zero `Buffer` usage, pure JS only

## References

- [deBridge DLN API Docs](https://docs.dln.trade/)
- [deBridge Chain IDs](https://docs.dln.trade/the-protocol/supported-chains)
