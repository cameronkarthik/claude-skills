---
name: ephemeral-wallet-pipeline
description: Use when building multi-wallet transaction pipelines with ephemeral keypairs -- token launches, bundled buys/sells, or any flow that generates temporary wallets, funds them, executes operations, and consolidates funds back. Covers the full lifecycle including error recovery and fund rescue.
version: 0.1.0
tools: Read, Grep, Glob, Bash, Edit, Write
---

# Ephemeral Wallet Pipeline

Complete lifecycle pattern for generating temporary Solana wallets, funding them from a parent wallet, executing operations (swaps, bridges, etc.), and consolidating funds back. Built from building a multi-wallet token bundler that lost real money when recovery flows were broken.

## When to Use

- Building a bundler or multi-wallet trading tool
- Any flow that creates temporary wallets for on-chain operations
- Token launch coordination across multiple wallets
- Obfuscated transaction flows (bridge wash, fund splitting)
- Any pipeline where funds leave a parent wallet and must return

## Architecture

```
Parent Wallet (user's Phantom/Solflare)
    |
    v  [Fund: single multi-transfer tx]
+----------+  +----------+  +----------+
| Wallet 1 |  | Wallet 2 |  | Wallet N |
+----------+  +----------+  +----------+
    |              |              |
    v              v              v
  [Execute operations: swaps, bridges, etc.]
    |              |              |
    v              v              v
+----------+  +----------+  +----------+
| Wallet 1 |  | Wallet 2 |  | Wallet N |
+----------+  +----------+  +----------+
    |
    v  [Consolidate: sweep all back]
Parent Wallet
```

## Critical Rule: Wallets Are Memory-Only

Ephemeral keypairs exist ONLY as JavaScript variables. They are:
- Generated at pipeline start
- Held in a Zustand store during the active session
- Destroyed when the user closes the tab or resets

**Never persist private keys to localStorage, IndexedDB, cookies, or any database.** If you need persistence across page refreshes during a long operation, use an encrypted IndexedDB vault with a wallet-signature-derived key (see the `solana-browser-wallet` skill).

## Wallet Generation

```typescript
import { Keypair } from "@solana/web3.js";

interface EphemeralWallet {
  publicKey: string;
  keypair: Keypair;
  label: string;
}

function generateEphemeralWallets(count: number): EphemeralWallet[] {
  if (count < 1 || count > 10) throw new Error("Wallet count must be 1-10");
  return Array.from({ length: count }, (_, i) => {
    const kp = Keypair.generate();
    return {
      publicKey: kp.publicKey.toBase58(),
      keypair: kp,
      label: `Wallet ${i + 1}`,
    };
  });
}
```

## Zustand Store Pattern

```typescript
import { createStore } from "zustand/vanilla";

interface WalletState {
  wallets: EphemeralWallet[];
  isActive: boolean;
  generateForTrade: (count: number) => EphemeralWallet[];
  getKeypair: (publicKey: string) => Keypair | null;
  clear: () => void;
}

const walletStore = createStore<WalletState>((set, get) => ({
  wallets: [],
  isActive: false,
  generateForTrade: (count) => {
    const wallets = generateEphemeralWallets(count);
    set({ wallets, isActive: true });
    return wallets;
  },
  getKeypair: (publicKey) => {
    return get().wallets.find((w) => w.publicKey === publicKey)?.keypair ?? null;
  },
  clear: () => set({ wallets: [], isActive: false }),
}));
```

## Pipeline State Machine

```typescript
enum PipelineStage {
  IDLE = "IDLE",
  FUNDING = "FUNDING",         // Parent -> ephemeral wallets
  BRIDGING_OUT = "BRIDGING_OUT", // Optional: obfuscate via bridge
  BUYING = "BUYING",           // Execute swaps on all wallets
  HOLDING = "HOLDING",         // User decides when to sell
  SELLING = "SELLING",         // Reverse swaps on all wallets
  BRIDGING_BACK = "BRIDGING_BACK", // Optional: bridge back
  CONSOLIDATING = "CONSOLIDATING", // Sweep all -> parent
  COMPLETE = "COMPLETE",
  ERROR = "ERROR",
}
```

### State Transitions

```
IDLE -> FUNDING -> BUYING -> HOLDING -> SELLING -> CONSOLIDATING -> COMPLETE
                      |                                  |
                      v                                  v
              BRIDGING_OUT -> BUYING           BRIDGING_BACK -> CONSOLIDATING
                      |                                  |
                      v                                  v
                    ERROR <-------- any stage --------ERROR
```

## Funding: Multi-Transfer Transaction

Fund all wallets in a single transaction to minimize confirmation waits:

```typescript
import {
  Transaction,
  SystemProgram,
  LAMPORTS_PER_SOL,
  PublicKey,
} from "@solana/web3.js";

function buildFundingTransaction(
  parentPubkey: PublicKey,
  wallets: EphemeralWallet[],
  amountsSOL: number[]
): Transaction {
  const tx = new Transaction();
  for (let i = 0; i < wallets.length; i++) {
    tx.add(
      SystemProgram.transfer({
        fromPubkey: parentPubkey,
        toPubkey: new PublicKey(wallets[i].publicKey),
        lamports: Math.floor(amountsSOL[i] * LAMPORTS_PER_SOL),
      })
    );
  }
  return tx;
}
```

### Amount Jitter (For Stealth)

If avoiding on-chain clustering detection, randomize amounts while preserving the total:

```typescript
function jitterAmounts(totalSOL: number, walletCount: number, jitterPct: number): number[] {
  const base = totalSOL / walletCount;
  const raw = Array.from({ length: walletCount }, () => {
    const factor = 1 + (Math.random() * 2 - 1) * (jitterPct / 100);
    return base * factor;
  });
  const sum = raw.reduce((a, b) => a + b, 0);
  return raw.map((v) => (v / sum) * totalSOL); // normalize to preserve total
}
```

## Consolidation: Sweep All Back

This is the most critical function. It MUST handle partial failures gracefully.

```typescript
async function consolidateToParent(
  wallets: EphemeralWallet[],
  parentPubkey: PublicKey,
  connection: Connection,
  onProgress: (pubkey: string, update: { error?: string; txSig?: string }) => void
): Promise<{ succeeded: number; failed: number }> {
  let succeeded = 0;
  let failed = 0;

  for (const w of wallets) {
    try {
      const balance = await connection.getBalance(w.keypair.publicKey);
      const fee = 5000; // tx fee in lamports
      const sendAmount = balance - fee;

      if (sendAmount <= 0) {
        onProgress(w.publicKey, {
          error: `Balance too low to recover (${(balance / LAMPORTS_PER_SOL).toFixed(6)} SOL)`,
        });
        failed++;
        continue;
      }

      const tx = new Transaction().add(
        SystemProgram.transfer({
          fromPubkey: w.keypair.publicKey,
          toPubkey: parentPubkey,
          lamports: sendAmount,
        })
      );

      const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash();
      tx.recentBlockhash = blockhash;
      tx.feePayer = w.keypair.publicKey;
      tx.sign(w.keypair);

      const sig = await connection.sendRawTransaction(tx.serialize(), {
        skipPreflight: false,
        maxRetries: 3,
      });

      await connection.confirmTransaction(
        { signature: sig, blockhash, lastValidBlockHeight },
        "confirmed"
      );

      onProgress(w.publicKey, { txSig: sig });
      succeeded++;
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      onProgress(w.publicKey, { error: msg });
      failed++;
    }
  }

  return { succeeded, failed };
}
```

## Error Recovery (Non-Negotiable)

These rules exist because real money was lost when they weren't followed.

### Rule 1: Cancel button is NEVER disabled

```tsx
// WRONG - user can't cancel during execution
<button disabled={executing} onClick={handleCancel}>Cancel</button>

// RIGHT - always clickable
<button onClick={handleCancel}>Cancel & Return Funds</button>
```

### Rule 2: Cancel/recover at ALL stages, not just one

```tsx
const showCancelButton =
  pipeline.stage !== PipelineStage.IDLE &&
  pipeline.stage !== PipelineStage.COMPLETE;

// Shows during FUNDING, BRIDGING_OUT, BUYING, HOLDING, SELLING,
// BRIDGING_BACK, CONSOLIDATING, AND ERROR
```

### Rule 3: Never filter out error wallets during recovery

```typescript
// WRONG - skips wallets that errored, leaving funds stranded
const walletsToRecover = wallets.filter((w) => !w.error);

// RIGHT - attempt recovery on ALL wallets that have keypairs
const walletsToRecover = wallets.filter((w) => w.keypair);
```

### Rule 4: Don't wipe state until recovery confirms

```typescript
// WRONG - immediately wipe on cancel
handleCancel = () => {
  consolidateToParent(wallets, parent, connection, onProgress);
  pipelineStore.reset(); // DESTROYS KEYPAIRS
  walletStore.clear();   // FUNDS ARE NOW UNRECOVERABLE
};

// RIGHT - only wipe after confirmed recovery
handleCancel = async () => {
  const { succeeded, failed } = await consolidateToParent(
    wallets, parent, connection, onProgress
  );
  if (succeeded > 0 && failed === 0) {
    pipelineStore.reset();
    walletStore.clear();
  } else if (succeeded > 0) {
    // Partial recovery - show warning, don't wipe remaining
    setWarning(`Recovered ${succeeded} wallets. ${failed} still have funds.`);
  } else {
    setError("Recovery failed on all wallets. Keys preserved.");
  }
};
```

### Rule 5: beforeunload warning during active pipeline

```typescript
useEffect(() => {
  if (pipeline.stage !== PipelineStage.IDLE && pipeline.stage !== PipelineStage.COMPLETE) {
    const handler = (e: BeforeUnloadEvent) => {
      e.preventDefault();
      e.returnValue = "Funds are in ephemeral wallets. Closing will lose access.";
    };
    window.addEventListener("beforeunload", handler);
    return () => window.removeEventListener("beforeunload", handler);
  }
}, [pipeline.stage]);
```

### Rule 6: Emergency key export as last resort

If bridge-back fails and funds are stuck on another chain, show private keys so the user can import into MetaMask/Phantom manually:

```tsx
{pipeline.error?.includes("intermediary chain") && evmWallets.length > 0 && (
  <div className="bg-yellow-950 border border-yellow-800 rounded-lg p-4">
    <h4>Emergency: EVM Wallet Keys</h4>
    <p>Funds are stuck on Base. Import these keys into MetaMask (Base network).</p>
    <p><strong>DO NOT refresh -- keys only exist in memory.</strong></p>
    {evmWallets.map((w, i) => (
      <div key={i}>
        <code>{w.address}</code>
        <button onClick={() => navigator.clipboard.writeText(w.privateKey)}>Copy Key</button>
      </div>
    ))}
  </div>
)}
```

## P&L Calculation

```typescript
interface PnLResult {
  grossProfit: number;  // returned - invested
  netProfit: number;    // grossProfit - totalFees
  totalFees: number;    // bridge + swap fees
  roi: number;          // (netProfit / invested) * 100
}

function calculatePnL(
  totalInvested: number,
  totalReturned: number,
  bridgeFees: number,
  swapFees: number
): PnLResult {
  const grossProfit = totalReturned - totalInvested;
  const totalFees = bridgeFees + swapFees;
  const netProfit = grossProfit - totalFees;
  const roi = totalInvested > 0 ? (netProfit / totalInvested) * 100 : 0;
  return { grossProfit, netProfit, totalFees, roi };
}
```

## Trade Persistence (Supabase)

Only log trades AFTER confirmed on-chain funding to avoid phantom records:

```typescript
// WRONG - creates record before anything is confirmed
const tradeId = await createTrade(supabase, userId, tradeData);
await fundWallets(); // might fail, leaving a garbage trade record

// RIGHT - create record after funding confirmed
const sig = await sendAndConfirmTransaction(fundingTx);
const tradeId = await createTrade(supabase, userId, {
  ...tradeData,
  fundingTxSig: sig,
  status: "funded",
});
```

## Integration Checklist

- [ ] Wallet generation: max 10 per pipeline, Keypair.generate() only
- [ ] Store: Zustand vanilla store, wallets in memory only
- [ ] Funding: single multi-transfer tx signed by parent wallet
- [ ] Amount jitter: if stealth mode, randomize while preserving total
- [ ] Pipeline state machine: clear stage transitions, ERROR reachable from any stage
- [ ] Cancel button: visible at ALL active stages, NEVER disabled
- [ ] Recovery: attempt on ALL wallets with keypairs (don't filter errors)
- [ ] State cleanup: only wipe after confirmed recovery, never before
- [ ] beforeunload: warn user during active pipeline
- [ ] Emergency export: show raw private keys if cross-chain recovery fails
- [ ] P&L: track invested/returned/fees, calculate ROI
- [ ] Trade records: create AFTER confirmed funding, not before
- [ ] Tests: use `// @vitest-environment node` for any file importing `@solana/web3.js`
